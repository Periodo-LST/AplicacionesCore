# Instalación de Longhorn

Guía para desplegar Longhorn en el clúster utilizando GitOps con ArgoCD y GitLab. Asume que ya tienes preparados `cert-manager` y el `ClusterIssuer` `step-ca-issuer`.

## Requisitos previos

- `kubectl` configurado contra el clúster.
- Repositorio de GitLab (`aplicacionescore.git`) registrado en ArgoCD.
- El issuer `step-ca-issuer` disponible en el clúster.

## Preparación de los nodos (en cada host del clúster)

### 1. Cargar el módulo `dm_crypt` y hacerlo persistente en el arranque

```bash
sudo modprobe dm_crypt
echo "dm_crypt" | sudo tee -a /etc/modules
```

### 2. Instalar las dependencias de almacenamiento e iniciar el servicio iSCSI

```bash
sudo apt-get update && sudo apt-get install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```

## Despliegue en ArgoCD

Aplica el manifiesto de la aplicación `longhorn-app.yaml` que combina el Helm Chart oficial con la fuente de GitLab:

```bash
kubectl apply -f longhorn-app.yaml -n argocd
```

> **Nota:** el contenido de `longhorn-app.yaml` (el `Application` de ArgoCD que referencia el chart oficial de Longhorn y los archivos `values.yaml` / `longhorn-cert.yaml` de tu repo GitLab) no se incluyó en el material original. Si lo necesitas, puedo ayudarte a construirlo.

## Verificación

```bash
kubectl get application longhorn-gitops -n argocd
kubectl get pods -n longhorn-system
kubectl get certificate -n longhorn-system
kubectl get secret -n longhorn-system longhorn-tls-v2
```

Si la aplicación en ArgoCD indica los estados `Synced` y `Healthy`, la interfaz web de Longhorn quedará disponible vía HTTPS a través de Traefik en `https://longhorn.fluffy.lst`.