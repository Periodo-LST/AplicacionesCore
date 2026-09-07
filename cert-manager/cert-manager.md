# Instalación y Configuración de cert-manager

Guía para desplegar `cert-manager` y el `ClusterIssuer` `step-ca-issuer` en el clúster utilizando GitOps con ArgoCD y GitLab.

## Requisitos previos

- `kubectl` configurado contra el clúster.
- Repositorio de GitLab (`aplicacionescore.git`) registrado en ArgoCD.
- Servicio `step-ca` operativo en el namespace `step-ca` y accesible por Traefik.

Sube la configuración a GitLab:

```bash
git add cert-manager/
git commit -m "feat: agregar valores y step-clusterissuer para cert-manager"
git push origin main
```

## Despliegue en ArgoCD

Aplica el manifiesto `cert-manager-app.yaml` que combina el Helm Chart oficial de Jetstack (`v1.14.4`) con la configuración de GitLab:

```bash
kubectl apply -f cert-manager-app.yaml -n argocd
```

## Configuración de Step-CA y Traefik

El `ClusterIssuer` utiliza `caBundle` y validación `http01` especificando `ingressClassName: traefik`. Esto permite a `cert-manager` validar los desafíos ACME con `step-ca` a través de Traefik de forma segura.

Si cambia el nombre de la clase de Ingress o la URL interna de `step-ca`, edita `cert-manager/step-clusterissuer.yaml` en GitLab.

## Comprobación y Verificación

Verifica que `cert-manager`, sus pods y el `ClusterIssuer` estén operativos:

```bash
kubectl get application cert-manager-gitops -n argocd
kubectl -n cert-manager get pods
kubectl get clusterissuer step-ca-issuer
```

Para validar el funcionamiento completo del flujo cert-manager -> step-ca, comprueba la emisión de un certificado del clúster (por ejemplo, en `cattle-system` o `longhorn-system`):

```bash
kubectl -n cattle-system get certificate tls-rancher-ingress
kubectl -n longhorn-system get certificate longhorn-tls-v2
```

Si el estado de los certificados pasa a `Ready` / `True`, el sistema de certificación está funcionando correctamente bajo el modelo GitOps.