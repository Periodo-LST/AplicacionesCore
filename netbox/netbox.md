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

## 9. Orden de despliegue

1. Namespace `netbox`.
2. PVCs (`netbox-postgres-pvc-v2`, `netbox-media-v2`).
3. Deployments/Services de PostgreSQL y Redis.
4. ConfigMap `netbox-internal-ca` con la CA de Keycloak.
5. Client OIDC creado en Keycloak (sección 6).
6. Aplicar el `Application` de ArgoCD → despliega NetBox con el `values.yaml` de la sección 7.

---

## 10. Verificación post-instalación

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
  "from django.conf import settings; print(settings.REMOTE_AUTH_ENABLED); print(settings.AUTHENTICATION_BACKENDS)"
```

Después, entra en `https://netbox.fluffy.lst/login/` y confirma que aparece el botón de login con Keycloak.

---

## 11. Errores encontrados durante esta instalación (y su causa)

| Síntoma | Causa | Solución |
|---|---|---|
| No aparece botón de SSO | `extraConfig` envuelto como `mi_archivo.py: \| <código python>` en vez de claves YAML planas — ese texto nunca se ejecuta | Usar `extraConfig[].values` como diccionario directo de settings |
| No aparece botón de SSO (2ª causa, simultánea) | `REMOTE_AUTH_ENABLED: false` y sin `REMOTE_AUTH_BACKEND` | `REMOTE_AUTH_ENABLED: true` + `REMOTE_AUTH_BACKEND` apuntando al backend OIDC |
| Backend no carga / import error | `social_core.backends.oidc.OIDCAuth` no existe | Usar `social_core.backends.open_id_connect.OpenIdConnectAuth` |
| `AuthConnectionError: SSLCertVerificationError` | El pod no confía en la CA interna que firma `keycloak.fluffy.lst` | Montar la CA vía ConfigMap + `extraVolumes`/`extraVolumeMounts` + `REQUESTS_CA_BUNDLE` |
| Ninguna variable de `extraEnv*` llega al pod | Nombre de clave incorrecto (`extraEnv`, luego `extraEnvVars`) | La clave real en `netbox-chart 5.0.10` es `extraEnvs` |