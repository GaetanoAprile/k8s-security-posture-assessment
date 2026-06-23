# Checklist Operativa per Cluster Kubernetes (Dev/Test)

Questa checklist definisce i requisiti minimi di sicurezza (Hardening) per il deployment di applicazioni in ambiente locale (Minikube/k3s) o di sviluppo, in conformità con i benchmark CIS e le best practice di sicurezza.

## 1. Gestione dei Privilegi (Pod Security)
- [ ] **Nessun container privilegiato:** Assicurarsi che `securityContext.privileged` sia impostato su `false` o omesso.
- [ ] **Blocco Escalation:** Impostare `allowPrivilegeEscalation: false` per impedire ai processi figli di ottenere permessi extra.
- [ ] **Esecuzione come Non-Root:** Il container non deve girare come amministratore. Impostare `runAsNonRoot: true` e definire un `runAsUser` specifico (es. 1000).
- [ ] **Drop delle Capabilities:** Rimuovere i privilegi di default di Linux impostando `capabilities.drop: ["ALL"]`.
- [ ] **File System in Sola Lettura:** Prevenire scritture non autorizzate sul disco impostando `readOnlyRootFilesystem: true`. Montare volumi `emptyDir` solo per cartelle strettamente necessarie (es. `/tmp`).

## 2. Isolamento (Network e Host)
- [ ] **Isolamento Rete Host:** Non agganciare il pod alla rete del nodo fisico (`hostNetwork: false`).
- [ ] **Isolamento Processi Host:** Non permettere al pod di vedere i processi del nodo (`hostPID: false` e `hostIPC: false`).
- [ ] **Esposizione Sicura dei Servizi:** Evitare l'uso di `type: NodePort` a meno che non sia strettamente necessario per il debug. Utilizzare `ClusterIP` e gestire l'accesso esterno tramite Ingress o port-forwarding locale.

## 3. Gestione Accessi (RBAC)
- [ ] **Stop al Default ServiceAccount:** Non utilizzare il `ServiceAccount` di default del namespace. Creare un ServiceAccount dedicato per l'applicazione.
- [ ] **Principio del Minimo Privilegio:** Non assegnare mai il `ClusterRole` di `cluster-admin` ai workload applicativi.

## 4. Gestione Risorse
- [ ] **Limiti Definiti:** Impostare sempre `resources.requests` e `resources.limits` (CPU e RAM) per prevenire attacchi DoS o colli di bottiglia nel cluster locale.