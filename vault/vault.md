# Vault

## Despliegue

### 1. Subir la configuración a GitLab

```bash
git add vault/values.yaml
git commit -m "feat: desplegar Vault vía GitOps con volumen de 2Gi"
git push origin main
```

### 2. Aplicar la Application en ArgoCD

```bash
kubectl apply -f vault-app.yaml -n argocd
argocd app sync vault-gitops
```

### 3. Inicializar Vault

El pod `vault-gitops-0` arranca en `0/1 Ready` hasta que se inicializa — es el comportamiento esperado, no un fallo.

```bash
kubectl exec -it vault-gitops-0 -n vault -- vault operator init
```

⚠️ Guarda las 5 Unseal Keys y el Root Token mostrados, en un gestor de secretos seguro (ver sección de Secrets más arriba).

### 4. Desprecintar (unseal) Vault

Aplica 3 de las 5 llaves generadas:

```bash
kubectl exec -it vault-gitops-0 -n vault -- vault operator unseal <UNSEAL_KEY_1>
kubectl exec -it vault-gitops-0 -n vault -- vault operator unseal <UNSEAL_KEY_2>
kubectl exec -it vault-gitops-0 -n vault -- vault operator unseal <UNSEAL_KEY_3>
```

Vault queda sellado (`Sealed`) cada vez que el pod se reinicia — **hay que repetir el unseal manualmente en cada reinicio**, salvo que se configure auto-unseal (ver "Notas de seguridad").

### 5. Verificación

```bash
kubectl get pvc -n vault
kubectl get pods -n vault
```

Accede a la UI en `https://vault.fluffy.lst`.

## Errores comunes

- **Pod en `0/1 Ready` indefinidamente tras el despliegue inicial**: es esperado — Vault no pasa el *readiness probe* hasta que está `unsealed`. No es un fallo, hace falta `vault operator init` + `vault operator unseal`.

- **Vault vuelve a `Sealed` tras cada reinicio del pod**: comportamiento normal sin auto-unseal configurado. Cada restart (actualización del chart, restart manual, reprogramación del pod) exige repetir el `unseal` a mano con 3 de las 5 llaves.

- **Conflicto de `selfHeal` con el `MutatingWebhookConfiguration` del injector**: si ves que Argo marca constantemente `OutOfSync` en `vault-gitops-agent-injector-cfg` sin que nadie lo edite, revisa que el bloque `ignoreDifferences` sobre `/webhooks/0/clientConfig/caBundle` siga presente en `vault-app.yaml` — el propio injector reescribe ese campo en tiempo real.

- **Necesitas reducir el PVC**: ver la sección de arriba — no se puede editar in situ, hay que recrear el `StatefulSet`/PVC (con o sin migración de datos según el caso).

## Notas de seguridad

- Las Unseal Keys y el Root Token **nunca deben versionarse en Git ni quedar en logs/historial de terminal**. Considera un gestor de secretos dedicado (el propio Vault, una vez desprecintado, puede gestionar sus propias claves; mientras tanto, usa algo como un gestor de contraseñas cifrado offline).
- Para producción, evalúa migrar de *Shamir unseal* (llaves manuales) a **auto-unseal** vía KMS (AWS KMS, GCP KMS, Transit de otro Vault, etc.) — elimina la necesidad de intervención manual tras cada reinicio y reduce el riesgo de perder las Unseal Keys.
- `replicas: 1` implica que no hay alta disponibilidad — un solo nodo Raft. Si necesitas HA, hay que pasar a `replicas: 3` (o más) con almacenamiento Raft distribuido, lo cual cambia sustancialmente el proceso de inicialización y unseal (join entre nodos).