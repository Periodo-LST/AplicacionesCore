## Instalación de cert-manager

Si quieres instalar cert-manager desde cero, la forma más simple y compatible con este repo es aplicar el manifiesto oficial de la release:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.2/cert-manager.yaml
```

Después verifica que los pods estén listos:

```bash
kubectl -n cert-manager get pods
```

Si prefieres Helm, asegúrate de instalar también los CRDs:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

## Aplicar el issuer de step-ca

Una vez cert-manager esté operativo, aplica el issuer que usa el repo:

```bash
kubectl apply -f step-ca/step-clusterissuer.yaml
```

Comprueba que queda creado:

```bash
kubectl get clusterissuer step-ca-issuer
```

## Nota sobre la configuración de step-ca

El issuer activo del repo usa `caBundle` y `http01` con `ingressClassName: traefik`. Eso encaja con la configuración actual de step-ca expuesta por Traefik y evita depender de una CA no confiable para cert-manager.

Si cambias la URL pública de `step-ca` o el nombre de la clase de ingress, actualiza también el issuer antes de dar por buena la instalación.

## Comprobación mínima

Para validar que el flujo está bien enlazado, revisa al menos esto:

```bash
kubectl -n cert-manager get pods
kubectl get clusterissuer step-ca-issuer
kubectl -n cattle-system get certificate tls-rancher-ingress
```

Si el certificado de Rancher pasa a estado `Ready`, el camino cert-manager -> step-ca está funcionando con la configuración del repo.