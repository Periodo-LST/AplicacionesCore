# Instalación de Traefik

Traefik se administra normalmente mediante la `Application` de Argo CD definida en `traefik-app.yaml`. Argo CD instala directamente el chart oficial de Traefik usando los valores incluidos en esa `Application`.

El fichero `traefik-values.yaml` solo se utiliza para una instalación manual alternativa con Helm. No es consumido por `traefik-app.yaml`.

## Requisitos previos

- `kubectl` configurado contra el clúster.
- `helm` instalado en el equipo desde el que vas a desplegar.
- El namespace `traefik-system` no necesita existir antes; Helm lo crea con `--create-namespace`.

## Instalación mediante Argo CD (recomendada)

Requiere que Argo CD ya esté instalado y que su credencial de GitLab esté configurada:

```bash
kubectl apply -f traefik/traefik-app.yaml
```

Argo CD creará el namespace `traefik-system` y desplegará Traefik con el chart y la configuración definidos en `traefik-app.yaml`.

## Verificación

```bash
kubectl get pods -n traefik-system
kubectl get svc -n traefik-system
kubectl get ds -n traefik-system
```

Si todo está correcto, Traefik quedará escuchando en los puertos `80` y `443` del host y podrá servir los Ingress del clúster.