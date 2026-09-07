# Instalación de Longhorn

Guía para desplegar Longhorn en el clúster utilizando GitOps con ArgoCD y GitLab. Asume que ya tienes preparados `cert-manager` y el `ClusterIssuer` `step-ca-issuer`.

## Requisitos previos

* `kubectl` configurado contra el clúster.
* Repositorio de GitLab (`aplicacionescore.git`) registrado en ArgoCD.
* El issuer `step-ca-issuer` disponible en el clúster.

## Preparación de los nodos (en cada host del clúster)

1. Cargar el módulo `dm_crypt` y hacerlo persistente en el arranque:

```bash
sudo modprobe dm_crypt
echo "dm_crypt" | sudo tee -a /etc/modules