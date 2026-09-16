# NetBox

Instalación de NetBox mediante el Helm chart oficial (`netbox-community/netbox-chart`), con PostgreSQL y Redis externos desplegados a mano, gestionado vía ArgoCD, y autenticación SSO contra Keycloak (OIDC) a través de una CA interna (step-ca).

## 1. CA interna de Keycloak (para validación TLS)

NetBox necesita confiar en la CA que firma el certificado de `keycloak.fluffy.lst`, o la llamada a `/.well-known/openid-configuration` falla con `SSLCertVerificationError`.

**Obtener el certificado de la CA** (la raíz o intermedia que firma el leaf cert de Keycloak, no necesariamente el `caBundle` del `ClusterIssuer` de ACME — verifícalo):

```bash
# Extrae la cadena completa tal como la sirve Keycloak
echo | openssl s_client -connect keycloak.fluffy.lst:443 -servername keycloak.fluffy.lst -showcerts 2>/dev/null \
  | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' > keycloak-chain.pem
```

**Crear el ConfigMap:**

```bash
kubectl create configmap netbox-internal-ca \
  --from-file=ca.crt=keycloak-chain.pem \
  -n netbox
```

**Verificar que esa CA valida correctamente antes de continuar:**

```bash
openssl s_client -connect keycloak.fluffy.lst:443 -servername keycloak.fluffy.lst \
  -CAfile keycloak-chain.pem </dev/null 2>&1 | grep "Verify return code"
# Debe devolver: Verify return code: 0 (ok)
```

---

## 2. Configuración del lado de Keycloak

En el realm que vayas a usar (en este ejemplo `Nutty`):

1. Crea un **Client** de tipo *OpenID Connect*, `Client ID: netbox`, con **Client authentication** activado (confidential).
2. **Valid Redirect URI**:
   ```
   https://netbox.fluffy.lst/complete/oidc/
   ```
3. **Web origins**: `https://netbox.fluffy.lst` (o `+` si quieres permitir todos los redirect URIs válidos).
4. Copia el **Client secret** (pestaña *Credentials*) — lo necesitarás en `SOCIAL_AUTH_OIDC_SECRET`.
5. Asegúrate de que el scope `email` esté asignado al client (normalmente viene por defecto).

---

## 3. Imagen personalizada con plugins

La imagen oficial de NetBox no trae plugins de terceros. Hace falta construir una imagen propia encima, publicarla en un registro, y apuntar el chart a ella.

### 3.1. Dockerfile

Desde NetBox 4.3+, la imagen oficial **no incluye `pip`** en el venv — hay que usar `uv` (ya viene instalado en la imagen, es lo que usan para construir el propio venv). También hay que pasarle un `--constraint` con el `requirements.txt` de NetBox para que `uv` no actualice Django/DRF a una versión incompatible al resolver las dependencias de los plugins (si no, puede romper NetBox con errores tipo `ImportError: cannot import name 'cc_delim_re'`).

```dockerfile
FROM netboxcommunity/netbox:v4.7.0

USER root

RUN /usr/local/bin/uv pip install \
    --python /opt/netbox/venv/bin/python3 \
    --constraint /opt/netbox/requirements.txt \
    --no-cache \
    netboxlabs-netbox-custom-objects \
    netbox-validity \
    netbox-lists \
    netbox-inventory \
    netbox-attachments \
    netbox-topology-views \
    netbox-reorder-rack \
    netbox-secrets \
    netbox-plugin-dns

# netbox_topology_views necesita esta carpeta estática, si no la crea sola
RUN mkdir -p /opt/netbox/netbox/static/netbox_topology_views/img

USER unit
```

> Si `/opt/netbox/requirements.txt` no existe en esa ruta exacta para tu tag de imagen, localízalo con:
> ```bash
> docker run --rm --entrypoint find netboxcommunity/netbox:v4.7.0 /opt/netbox -maxdepth 1 -name "*requirements*.txt"
> ```

**Tabla de paquete pip → nombre de módulo** (no siempre coinciden):

| Módulo (`PLUGINS`) | Paquete pip |
|---|---|
| `netbox_custom_objects` | `netboxlabs-netbox-custom-objects` |
| `validity` | `netbox-validity` |
| `netbox_lists` | `netbox-lists` |
| `netbox_inventory` | `netbox-inventory` |
| `netbox_attachments` | `netbox-attachments` |
| `netbox_topology_views` | `netbox-topology-views` |
| `netbox_reorder_rack` | `netbox-reorder-rack` |
| `netbox_secrets` | `netbox-secrets` |
| `netbox_dns` | `netbox-plugin-dns` (el nombre pip cambió recientemente de `netbox-dns`) |

### 3.2. Pipeline de CI (GitLab) para construir y publicar la imagen

Usa el **Container Registry propio del proyecto en GitLab** — no hace falta montar un registro aparte. El job se autentica solo con `CI_JOB_TOKEN` (no hacen falta variables de CI/CD manuales):

`.gitlab-ci.yml` (en la raíz del repo; si el repo ya tiene pipeline, usa `include:` en vez de sobrescribir):

```yaml
stages:
  - build

build-netbox-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    TAG: v4.7.0-plugins-${CI_COMMIT_SHORT_SHA}
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - echo "$CI_JOB_TOKEN" | docker login $CI_REGISTRY -u gitlab-ci-token --password-stdin
  script:
    - docker build -t ${CI_REGISTRY_IMAGE}/netbox-custom:${TAG} -f netbox/docker/Dockerfile netbox/docker
    - docker push ${CI_REGISTRY_IMAGE}/netbox-custom:${TAG}
  rules:
    - changes:
        - netbox/docker/Dockerfile
```

El tag generado (`v4.7.0-plugins-<sha>`) se ve en los logs del job o en `Proyecto → Build → Registry`.

⚠️ Con `pullPolicy: IfNotPresent`, si reutilizas el mismo tag entre builds, Kubernetes no vuelve a descargar la imagen nueva aunque hayas hecho push. Usa siempre un tag que incluya el SHA del commit (como arriba).

### 3.3. Acceso de Kubernetes al registro (`imagePullSecrets`)

El `CI_JOB_TOKEN` del pipeline solo vive dentro del job — el **pod** de NetBox necesita sus propias credenciales persistentes para hacer `pull`, porque el Container Registry de GitLab es privado.

1. Crea un **Deploy Token** con scope `read_registry`: `Proyecto → Settings → Repository → Deploy tokens`.
2. Crea el secret en el namespace `netbox`:
   ```bash
   kubectl create secret docker-registry netbox-registry-creds \
     --docker-server=<host:puerto-del-registro> \
     --docker-username=<usuario-del-deploy-token> \
     --docker-password=<token-generado> \
     -n netbox
   ```

### 3.4. `values.yaml` — bloque de imagen

⚠️ En `netbox-chart 5.0.10`, `image` separa **registro** y **repositorio** en dos campos (por defecto `registry: ghcr.io`). Si metes el hostname completo dentro de `repository`, el chart lo concatena con el registro por defecto y genera un nombre de imagen inválido (`InvalidImageName`). El campo de pull secrets tampoco es `imagePullSecrets` a nivel raíz, sino `image.pullSecrets`:

```yaml
image:
  registry: gitlab.lst.tfo.upm.es:4567
  repository: kubernetescore/aplicacionescore/netbox-custom
  tag: v4.7.0-plugins-<sha-del-commit>
  pullPolicy: IfNotPresent
  pullSecrets:
    - netbox-registry-creds
```

---

## 4. Activación de plugins (`PLUGINS`)

Se declaran dentro de `extraConfig[].values`, igual que el resto de settings de NetBox:

```yaml
extraConfig:
  - values:
      # ... (settings de OIDC, etc.)

      PLUGINS:
        - 'netbox_attachments'
        - 'netbox_topology_views'
        - 'netbox_secrets'
        - 'netbox_reorder_rack'
        - 'netbox_lists'
        - 'netbox_inventory'
        - 'netbox_custom_objects'
        - 'validity'
        - 'netbox_dns'

      # Solo necesario si usas el validador de Data Source de "validity"
      CUSTOM_VALIDATORS:
        core.datasource:
          - "validity.custom_validators.DataSourceValidator"
```

### ⚠️ No actives todos los plugins nuevos a la vez la primera vez

Es un patrón de fallo conocido en NetBox: con muchos plugins nuevos simultáneos, si algo va mal el servidor puede crashear sin decir cuál es el culpable. Actívalos en tandas pequeñas (2-3 a la vez), y repite este ciclo después de cada cambio en `PLUGINS`:

```bash
kubectl exec -it -n netbox deploy/netbox-gitops -- python /opt/netbox/netbox/manage.py migrate
kubectl rollout restart deployment/netbox-gitops -n netbox
kubectl rollout restart deployment/netbox-gitops-worker -n netbox
kubectl get pods -n netbox -w
```

Verifica que cargaron bien antes de añadir la siguiente tanda:

```bash
kubectl exec -it -n netbox deploy/netbox-gitops -- \
  python /opt/netbox/netbox/manage.py shell -c "
from django.conf import settings
print(settings.PLUGINS)
"
```

---

## 5. NetBox 4.5.6 → 4.7.0 (para poder usar `netbox_attachments`)

`netbox_attachments` requiere NetBox ≥4.7.0. Esto obligó a subir de versión desde el 4.5.6 inicial.

**Antes de migrar:**

```bash
kubectl exec -n netbox deploy/netbox-postgres -- \
  pg_dump -U netbox netbox > netbox-backup-pre-4.7.sql
```

**Cosas que cambia NetBox 4.7:**
- Deja de soportar PostgreSQL 14 (no afecta, se usa `postgres:15-alpine`).
- Necesita la extensión `ltree` en PostgreSQL — se crea automáticamente en la migración si el usuario de BD tiene privilegios (el usuario `POSTGRES_USER` de la imagen oficial de Postgres es superusuario por defecto, así que no hace falta nada extra).
- Deja de soportar Redis 5.x (no afecta, se usa `redis:7-alpine`).

El pod principal aplica las migraciones automáticamente al arrancar, pero **el worker no** — si el worker se queda en `CrashLoopBackOff` con errores tipo `column ... does not exist`, ejecuta `migrate` a mano (ver comando en la sección 4) y reinicia el worker.

---

## 6. Orden de despliegue

1. Namespace `netbox`.
2. PVCs (`netbox-postgres-pvc-v2`, `netbox-media-v2`).
3. Deployments/Services de PostgreSQL y Redis.
4. ConfigMap `netbox-internal-ca` con la CA de Keycloak.
5. Client OIDC creado en Keycloak (sección 2).
6. Imagen personalizada con plugins construida y publicada (sección 3), y secret `netbox-registry-creds` creado.
7. Aplicar el `Application` de ArgoCD → despliega NetBox con el `values.yaml`.
8. Activar plugins en tandas (sección 4).

---

## 7. Verificación post-instalación

```bash
# El deployment se llama <release>-gitops si el Application/release se llama así,
# revisa el nombre real con:
kubectl get deploy -n netbox

# Confirma que las env vars llegaron al pod
kubectl exec -n netbox deploy/netbox-gitops -- env | grep -E "REQUESTS_CA_BUNDLE|ALLOWED_HOSTS"

# Confirma que el certificado de la CA está montado
kubectl exec -n netbox deploy/netbox-gitops -- cat /etc/ssl/certs/step-ca.crt

# Confirma que la CA valida el certificado de Keycloak desde dentro del pod
kubectl exec -n netbox deploy/netbox-gitops -- \
  openssl s_client -connect keycloak.fluffy.lst:443 -servername keycloak.fluffy.lst \
  -CAfile /etc/ssl/certs/step-ca.crt </dev/null 2>&1 | grep "Verify return code"

# Confirma los settings de Django cargados realmente
kubectl exec -n netbox deploy/netbox-gitops -- \
  python /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(settings.REMOTE_AUTH_ENABLED); print(settings.AUTHENTICATION_BACKENDS); print(settings.PLUGINS)"
```

Después, entra en `https://netbox.fluffy.lst/login/` y confirma que aparece el botón de login con Keycloak. En `Admin → Plugins` confirma que los plugins activados aparecen sin errores (ten en cuenta que las columnas `Active`/`Local`/`Certified` de esa tabla se muestran como iconos, que a veces no se copian bien como texto al seleccionar/pegar).

---

## 8. Errores encontrados durante esta instalación (y su causa)

| Síntoma | Causa | Solución |
|---|---|---|
| No aparece botón de SSO | `extraConfig` envuelto como `mi_archivo.py: \| <código python>` en vez de claves YAML planas — ese texto nunca se ejecuta | Usar `extraConfig[].values` como diccionario directo de settings |
| No aparece botón de SSO (2ª causa, simultánea) | `REMOTE_AUTH_ENABLED: false` y sin `REMOTE_AUTH_BACKEND` | `REMOTE_AUTH_ENABLED: true` + `REMOTE_AUTH_BACKEND` apuntando al backend OIDC |
| Backend no carga / import error | `social_core.backends.oidc.OIDCAuth` no existe | Usar `social_core.backends.open_id_connect.OpenIdConnectAuth` |
| `AuthConnectionError: SSLCertVerificationError` | El pod no confía en la CA interna que firma `keycloak.fluffy.lst` | Montar la CA vía ConfigMap + `extraVolumes`/`extraVolumeMounts` + `REQUESTS_CA_BUNDLE` |
| Ninguna variable de `extraEnv*` llega al pod | Nombre de clave incorrecto (`extraEnv`, luego `extraEnvVars`) | La clave real en `netbox-chart 5.0.10` es `extraEnvs` |
| `docker build`: `No module named pip` | Desde NetBox 4.3+ la imagen no trae `pip` en el venv | Usar `uv pip install --python /opt/netbox/venv/bin/python3` |
| `ImportError: cannot import name 'cc_delim_re'` | `uv` actualizó Django/DRF a una versión incompatible al resolver dependencias de un plugin | Instalar con `--constraint /opt/netbox/requirements.txt` |
| Pod `InvalidImageName` | Se metió el hostname del registro dentro de `image.repository`; el chart lo concatena con `image.registry` (por defecto `ghcr.io`) | Separar `image.registry` y `image.repository`, y usar `image.pullSecrets` en vez de `imagePullSecrets` a nivel raíz |
| `ImagePullBackOff` | Pod sin credenciales para un registro privado | Crear Deploy Token + secret `docker-registry` + referenciarlo en `image.pullSecrets` |
| `Unable to load plugin X: requires NetBox minimum version Y` | El plugin declara un rango de versiones de NetBox incompatible con la instalada | Actualizar NetBox si el plugin lo requiere (mínimo), o esperar a que el plugin publique soporte (máximo) — no se autoactualiza |
| Worker en `CrashLoopBackOff` con `column ... does not exist` tras subir de versión | El pod principal migra la BD al arrancar, pero el worker no | `python manage.py migrate` a mano, luego reiniciar el worker |
| `helm pull` falla con `500 Internal Server Error` al sincronizar ArgoCD | Fallo puntual de GitHub Releases (el repo del chart redirige ahí) | Reintentar / hard refresh en ArgoCD; si persiste, vendorizar el chart en el propio repo |