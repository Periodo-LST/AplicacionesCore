# Rancher — Despliegue GitOps con ArgoCD

## Instrucciones de despliegue

### 1. Subir los cambios al repositorio remoto

```bash
git add rancher/
git commit -m "feat: configuracion de despliegue de rancher en gitops"
git push origin HEAD
```

### 2. Aplicar la Application en ArgoCD

```bash
kubectl apply -f rancher/application.yaml
```

### 3. Verificar el estado de la sincronización

```bash
kubectl get application -n argocd rancher-gitops
kubectl get pods -n cattle-system
```

### 4. Extraer la contraseña de administrador inicial

Si no definiste `bootstrapPassword` en `values.yaml`, Rancher la generó aleatoriamente y la guardó en un Secret:

```bash
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

El comando devuelve la contraseña en texto plano directamente en pantalla. Úsala para el primer login en la UI de Rancher y **cámbiala de inmediato** desde la propia interfaz.

## Verificación

```bash
kubectl get pods -n cattle-system
kubectl get ingress -n cattle-system
```

Accede a la UI en `<TODO: https://xxx o el hostname real>` y completa el asistente inicial (cambio de contraseña, URL del servidor, etc.).

`<TODO>` Si Rancher usa un certificado emitido por `step-ca-issuer`, valida también:

```bash
kubectl -n cattle-system get certificate tls-rancher-ingress
```

Debe quedar en `Ready: True` (ver `cert-manager.md` para troubleshooting si no es así).

## Errores comunes

`<TODO: completar con lo que vayas encontrando durante el despliegue real, siguiendo el mismo formato que step-ca.md y cert-manager.md>`

## Notas de seguridad

- La contraseña de `bootstrap-secret` es de un solo uso para el primer login — cámbiala inmediatamente desde la UI tras el primer acceso.
- Si fijas `bootstrapPassword` a mano en `values.yaml`, no la dejes en texto plano versionada en Git sin cifrar (Sealed Secrets / External Secrets Operator + Vault — mismo criterio que en `step-ca.md` y `vault.md`).