# Report Misconfigurazioni Intenzionali (Fase 1)

Per testare l'efficacia dei tool di auditing (kube-bench e kubeaudit) e simulare un ambiente a rischio, sono state introdotte intenzionalmente **7 vulnerabilità critiche** nel manifest iniziale `vulnerable-demo.yaml`. 

Queste coprono le tre macro-aree principali di sicurezza di Kubernetes:

## A. Gestione Accessi (RBAC)
1. **RBAC Permissivo:** Assegnazione del ruolo di `cluster-admin` (poteri assoluti sull'intero cluster) al `ServiceAccount` di default tramite un `ClusterRoleBinding` insicuro.

## B. Esposizione di Rete
2. **Esposizione Insegura:** Servizio esposto direttamente sulle porte del nodo fisico utilizzando `type: NodePort` (sulla porta 30080), bypassando le restrizioni di un Ingress controller e aprendo un potenziale varco dall'esterno.

## C. Sicurezza dei Pod (Pod Security)
3. **Container Privilegiato:** Esecuzione del container con il flag `privileged: true`, disabilitando i meccanismi di isolamento standard (cgroups/namespaces) e dando all'app gli stessi poteri del nodo host.

4. **Esecuzione come Root:** Container in esecuzione con i privilegi massimi di sistema amministratore (`runAsUser: 0`).

5. **Condivisione Rete Host:** Aggancio diretto alla rete fisica del nodo ospitante (`hostNetwork: true`), permettendo potenzialmente di intercettare o manipolare il traffico dell'intero nodo.

6. **Condivisione Processi Host:** Visibilità completa dei processi in esecuzione sul nodo fisico (`hostPID: true`), con il rischio di keylogging o furto di credenziali dalla memoria di altri applicativi.

7. **Risorse Illimitate:** Mancanza di direttive `requests` e `limits` per CPU e memoria, esponendo il cluster a potenziali rischi di saturazione e attacchi Denial of Service (DoS) in caso di compromissione.