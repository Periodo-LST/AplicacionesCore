# step-ca — Despliegue GitOps con ArgoCD

## Requisitos previos

- Cluster de Kubernetes con ArgoCD instalado, y `kubectl` configurado contra él.
- cert-manager instalado (usa el `ClusterIssuer` descrito en `cert-manager.md` para renovar certificados vía ACME).
- Traefik como IngressController, con el provider `kubernetescrd` habilitado (usamos el CRD `IngressRouteTCP`).
- StorageClass `longhorn` disponible (o ajustar `values.yaml`).
- Archivos de la CA disponibles localmente: `root.crt`, `intermediate.crt`, `intermediate.key` — **deben pertenecer todos a la misma cadena** (ver nota de la sección "Errores comunes" más abajo).
- Un servidor DNS interno (fuera del cluster) donde agregar el hostname público de la CA.

## Pasos para recrear el despliegue

### 1. Crear el namespace

```bash
kubectl create namespace step-ca
```

### 2. Crear los Secrets con material de la CA

> ⚠️ **Importante:** `root.crt` e `intermediate.crt` tienen que pertenecer a la misma cadena. Verificalo antes de cargar nada:
> ```bash
> openssl verify -CAfile root.crt intermediate.crt
> # debe imprimir: intermediate.crt: OK
> ```

```bash
# Certificados públicos (root + intermedio)
kubectl create secret generic step-ca-gitops-step-certificates-certs \
  -n step-ca \
  --from-file=root_ca.crt=root.crt \
  --from-file=intermediate_ca.crt=intermediate.crt

# Clave privada del intermedio (cifrada)
kubectl create secret generic step-ca-gitops-step-certificates-secrets \
  -n step-ca \
  --from-file=intermediate_ca_key=intermediate.key

# Password para descifrar la clave anterior
kubectl create secret generic step-ca-gitops-step-certificates-ca-password \
  -n step-ca \
  --type=smallstep.com/ca-password \
  --from-literal=password='<PASSWORD_CA>'

# Password del provisioner JWK (descifra el "encryptedKey" del ca.json)
kubectl create secret generic step-ca-gitops-step-certificates-provisioner-password \
  -n step-ca \
  --from-literal=password='<PASSWORD_PROVISIONER>'
```

### 3. Crear el ConfigMap de configuración

```bash
kubectl apply -f step-ca-config.yaml
```

Contiene `ca.json` (autoridad, provisioners, DB, TLS) y `defaults.json` (defaults del CLI `step`). El `ca.json` referencia las rutas donde el chart monta cada volumen:

```json
{
  "root": "/home/step/certs/root_ca.crt",
  "crt": "/home/step/certs/intermediate_ca.crt",
  "key": "/home/step/secrets/intermediate_ca_key",
  ...
}
```

> ⚠️ El campo `"key"` apunta al volumen `secrets` (`/home/step/secrets/...`), **no** al volumen `certs`. Es un error común confundirlos.

### 4. Desplegar la Application de ArgoCD

```bash
kubectl apply -f step-ca-app.yaml
```

ArgoCD sincroniza el chart con los `existingSecrets` apuntando a los recursos creados en los pasos 2-3. Verificá:

```bash
kubectl get application step-ca-gitops -n argocd \
  -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}'
```

### 5. Exponer el servicio (TLS passthrough)

```bash
kubectl apply -f step-ca-ingress.yaml
```

step-ca sirve su propio TLS (es la CA) — Traefik solo enruta por SNI sin descifrar (`tls.passthrough: true`). **No uses un `Ingress` normal para esto**: Traefik terminaría el TLS con su propio certificado y rompería la cadena de confianza del cliente ACME.

### 6. DNS

Agregá una entrada en tu DNS interno (fuera del cluster) apuntando el hostname público de la CA a la IP que expone tu Ingress Controller (firewall/virtual switch/LoadBalancer, según tu topología):

```
step-ca.fluffy.lst  →  <IP externa de Traefik>
```

Adicionalmente, para que los clientes ACME que corren **dentro** del cluster (como cert-manager) no dependan del firewall/DNS externo, hay un override en CoreDNS que resuelve el mismo hostname directo a la IP interna del Service:

```bash
kubectl edit configmap rke2-coredns-rke2-coredns -n kube-system
```

```
    hosts {
    <ClusterIP del Service step-ca-gitops-step-certificates>  step-ca.fluffy.lst
    fallthrough
    }
```

(reemplaza CoreDNS por su equivalente si no usás RKE2). El plugin `reload` de CoreDNS recarga solo, sin reiniciar pods. Verificá la resolución desde dentro del cluster:

```bash
kubectl -n step-ca run dns-test --rm -it --image=busybox --restart=Never -- \
  nslookup step-ca.fluffy.lst
```

### 7. Desplegar el ClusterIssuer de cert-manager

Ver `cert-manager.md` — el `ClusterIssuer` vive en ese repo, no en este.

```bash
kubectl apply -f cert-manager/step-clusterissuer.yaml
kubectl get clusterissuer step-ca-issuer -w
```

### 8. Verificar

```bash
kubectl get pods,svc -n step-ca
kubectl logs step-ca-gitops-step-certificates-0 -n step-ca
kubectl get certificates -A   # certificados de cert-manager deberían renovar solos
```

## Solicitud de certificados

### Desde un cliente `step` (bootstrap + certificado manual)

```bash
step ca bootstrap --ca-url https://step-ca.fluffy.lst --fingerprint <FINGERPRINT_ROOT_CRT>
step ca certificate "mi-servicio-externo.upm.es" certificado.crt certificado.key \
  --provisioner acme-internal
```

### Vía ACME (certbot)

```bash
certbot certonly --standalone \
  --email it@lst.tfo.upm.es \
  --server https://step-ca.fluffy.lst/acme/acme-internal/directory \
  -d mi-servicio-externo.upm.es
```

### Vía ACME (Proxmox VE)

Proxmox tiene su propio cliente ACME integrado, no usa certbot:

```bash
pvenode acme account register acme-internal tu-email@dominio.com \
  --directory https://step-ca.fluffy.lst/acme/acme-internal/directory
pvenode acme cert order --force
```

### Vía cert-manager (dentro del cluster)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mi-servicio-tls
  namespace: mi-namespace
spec:
  secretName: mi-servicio-tls
  dnsNames:
    - mi-servicio.fluffy.lst
  issuerRef:
    name: step-ca-issuer
    kind: ClusterIssuer
```

## Errores comunes (aprendidos a las malas)

- **`accountDoesNotExist` en un cliente ACME externo**: la base de datos de step-ca (Badger, en el PVC) es donde vive el registro de cuentas ACME. Si el volumen persistente se recrea o se pierde, todas las cuentas ACME externas quedan huérfanas — cada cliente (Proxmox, certbot, etc.) necesita **re-registrar su cuenta** contra el mismo `directory`, no solo reintentar la renovación.

- **`certificate signed by unknown authority`**: el `root.crt` usado en el `caBundle` del `ClusterIssuer` (y en el Secret `-certs`) no es el mismo que firmó el `intermediate.crt` cargado. Verificar siempre con `openssl verify -CAfile root.crt intermediate.crt` antes de cargar nada — es fácil mezclar archivos de distintas generaciones de la CA.

- **`authority.GetTLSCertificate: DNS name "X" is not permitted by any constraint`** (step-ca crashea en bucle al arrancar): se intentó añadir a `dnsNames` en `ca.json` un nombre que la CA raíz no puede firmar por sus *Name Constraints* (ver más abajo). El pod nunca llega a levantar — hay que quitar ese nombre de `dnsNames` inmediatamente para recuperar el servicio, y resolver el acceso por otra vía (DNS/hostname permitido) en vez de forzar el SAN.

- **ConfigMap/Secret que "no se actualiza"**: si `bootstrap.configmaps` o `bootstrap.secrets` quedan en `true` (el default), el chart sigue gestionando esos recursos como placeholders vacíos, y ArgoCD (`selfHeal: true`) revierte cualquier `kubectl apply` manual sobre ellos. Hay que poner explícitamente `bootstrap.configmaps: false` y `bootstrap.secrets: false` junto con `bootstrap.enabled: false`.

- **`must be unique` en volumeMounts**: el chart ya monta `/home/step/config` de forma incondicional — no agregues `extraVolumes`/`extraVolumeMounts` manuales a esa misma ruta.

- **`"key"` apuntando mal en `ca.json`**: la clave privada del intermedio vive en el volumen `secrets` (`/home/step/secrets/intermediate_ca_key`), no en `certs`.

- **El `ClusterIssuer` funciona a ratos, luego falla, luego vuelve a funcionar, sin que nadie toque nada**: antes de sospechar de step-ca, comprueba que no haya una instalación de cert-manager duplicada compitiendo con la buena — es la causa más probable de comportamiento intermitente e inconsistente. Ver la sección de errores comunes en `cert-manager.md`.

- **Redimensionar el PVC de la base de datos (`database-<statefulset>-0`)**: Kubernetes no permite reducir un PVC por la API estándar (solo expandir). Para achicarlo hace falta migrar los datos a mano a un PVC nuevo con un pod ayudante (`cp -a` entre dos volúmenes montados) y actualizar `persistence.size` en `values.yaml` para que coincida exactamente con el PVC final — si no coincide, el `volumeClaimTemplate` del StatefulSet (campo inmutable) rechazará el próximo sync.

## Notas de seguridad

- Los Secrets con material criptográfico (`-certs`, `-secrets`, `-ca-password`, `-provisioner-password`) se crean manualmente y **no están en este repositorio en texto plano**. Para un flujo 100% GitOps, considerar Sealed Secrets o External Secrets Operator + Vault.
- El intermedio de esta CA tiene *Name Constraints* — no puede firmar para dominios fuera de lo permitido (por ejemplo, no puede emitir para `*.svc.cluster.local`). Esto es intencional y **no debe removerse** solo para simplificar el DNS interno; en su lugar, usa un hostname permitido (`*.lst`, `*.lst.tfo.upm.es`) resuelto vía CoreDNS a la IP interna correspondiente (ver sección DNS).