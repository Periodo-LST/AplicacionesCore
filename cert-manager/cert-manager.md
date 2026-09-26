# cert-manager — Despliegue GitOps con ArgoCD

## Requisitos previos

- `kubectl` configurado contra el clúster.
- Repositorio de GitLab (`xxx`) registrado en ArgoCD.
- Servicio `step-ca` operativo en el namespace `step-ca`, con su `ClusterIssuer` apuntando al hostname correcto (ver `step-ca.md`).
- Traefik como IngressController (`ingressClassName: traefik`), usado por el solver `http01`.

## 1 Generar el `caBundle` correctamente

El `caBundle` es el certificado raíz de step-ca (`root_ca.crt`), en base64, de una sola línea, sin saltos de línea intermedios. Se extrae directamente del Secret real de step-ca — **nunca lo copies/pegues a mano de una terminal**, es la forma más fácil de corromperlo (nos pasó):

```bash
kubectl -n step-ca get secret step-ca-gitops-step-certificates-certs \
  -o jsonpath='{.data.root_ca\.crt}' | base64 -d > root_ca.pem

# validar ANTES de usarlo
openssl x509 -in root_ca.pem -noout -text

# generar el valor para caBundle, en una sola línea
base64 -w0 root_ca.pem
```

Pega el resultado en `spec.acme.caBundle` del `ClusterIssuer`. Si `openssl x509` falla al parsear el PEM, no sigas — el problema está en el certificado, no en cert-manager.

## 2 Despliegue en ArgoCD

Sube la configuración a GitLab:

```bash
git add cert-manager/
git commit -m "feat: agregar valores y step-clusterissuer para cert-manager"
git push origin main
```

Aplica la Application:

```bash
kubectl apply -f cert-manager/cert-manager-app.yaml -n argocd
```

## 3 Configuración de step-ca y Traefik

El `ClusterIssuer` usa `caBundle` y validación `http01` con `ingress.class: traefik`. Esto permite a cert-manager validar los desafíos ACME contra step-ca a través de Traefik.

Si cambia la clase de Ingress o la URL de step-ca, edita `cert-manager/step-clusterissuer.yaml` en GitLab — **nunca edites el `ClusterIssuer` a mano en el clúster**: con `selfHeal: true`, Argo lo revierte al estado de Git en cualquier momento.

## Comprobación y verificación

```bash
kubectl get application cert-manager-gitops -n argocd
kubectl -n cert-manager get pods
kubectl get clusterissuer step-ca-issuer
kubectl describe clusterissuer step-ca-issuer   # el Status.Conditions debe decir Ready: True
```

Para validar el flujo completo cert-manager → step-ca, emite un certificado de prueba **con un dominio real** que resuelva (nunca uses un nombre inventado, el self-check HTTP-01 fallará sin más):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mi-servicio-tls
  namespace: mi-namespace
spec:
  secretName: mi-servicio-tls
  dnsNames:
    - xxx   # debe caer bajo las Name Constraints configuradas en step-ca
  issuerRef:
    name: step-ca-issuer
    kind: ClusterIssuer
```

```bash
kubectl describe certificate mi-servicio-tls -n mi-namespace
```

También puedes revisar certificados ya emitidos de otros componentes del clúster:

```bash
kubectl -n cattle-system get certificate tls-rancher-ingress
kubectl -n longhorn-system get certificate longhorn-tls-v2
```

Si el estado pasa a `Ready: True`, el sistema de certificación funciona correctamente bajo GitOps.

## Errores comunes (aprendidos a las malas)

- **`caBundle` corrupto** (`cert bundle didn't contain any valid certificates`): el base64 se corrompió al copiar/pegar por terminal (un carácter de control o salto de línea mal puesto rompe el PEM interno). Solución: generarlo siempre con `kubectl get secret ... | base64 -d | openssl x509 -noout -text` para validar, y `base64 -w0` para producir el valor final — nunca a mano.

- **`x509: certificate is valid for X, not Y`**: el `server` del `ClusterIssuer` usa un hostname (normalmente el Service interno `*.svc.cluster.local`) que **no coincide con ningún SAN** del certificado de servidor de step-ca. La causa de fondo suele ser que la CA raíz de step-ca tiene *Name Constraints* que impiden emitir certificados para `*.svc.cluster.local` (ver `step-ca.md`) — la solución **no** es añadir ese SAN (la CA lo rechazará y step-ca directamente crasheará al arrancar con `DNS name ... is not permitted by any constraint`), sino apuntar el `server` al hostname externo permitido (`xxx`) y asegurarse de que resuelve también **dentro** del clúster (vía CoreDNS, ver sección DNS de `step-ca.md`).

- **`connect: connection refused` intermitente, incluso con todo bien configurado**: revisa si hay **más de una instalación de cert-manager corriendo en el mismo namespace**. Es fácil terminar con un release de Helm suelto (instalado antes de migrar a GitOps, con un `meta.helm.sh/release-name` distinto al de la Application de Argo) compitiendo con la instalación buena por el mismo `ClusterIssuer`. Diagnóstico:
  ```bash
  kubectl -n cert-manager get pods
  # busca pods que NO tengan el sufijo -gitops (o el nombre que use tu Application)
  helm -n cert-manager list
  ```
  Si aparece un release huérfano y roto, desinstálalo (las CRDs se conservan automáticamente, son compartidas):
  ```bash
  helm -n cert-manager uninstall <release-viejo>
  ```
  Esto puede causar horas de comportamiento errático e intermitente (el `ClusterIssuer` parece arreglarse y luego "se rompe solo") si no se detecta a tiempo — conviene comprobarlo como primer paso siempre que el comportamiento sea inconsistente entre reintentos.

- **`IssuerNotReady` justo después de que el `ClusterIssuer` pasó a `Ready`**: normalmente una condición de carrera de caché del controlador — reintenta al cabo de unos segundos, o borra y recrea el `CertificateRequest`/`Certificate` para forzar una reconciliación limpia.

- **`Challenge` atascado en `pending` con `no such host`**: el dominio del `Certificate` de prueba no resuelve por DNS. No es un fallo de cert-manager ni de step-ca — usa siempre un dominio real y resoluble para las pruebas.

## Notas de seguridad

- No captures nunca el `caBundle` ni ningún certificado con `cat`/copy-paste a través de una terminal con wrapping automático de línea — usa siempre pipes (`kubectl get secret ... -o jsonpath=... | base64 -d`) para evitar corrupción silenciosa.
- Verifica periódicamente que no haya instalaciones de cert-manager duplicadas en el clúster (`helm -n cert-manager list`), sobre todo tras migraciones de una instalación manual a GitOps.