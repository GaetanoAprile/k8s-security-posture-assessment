# Standard Operativo e Istruzioni di Auditing (Fase 5)

Questa documentazione fornisce le linee guida definitive (*Golden Path*) per il rilascio di workload sicuri all'interno del cluster e le istruzioni procedurali per l'esecuzione di una demo di auditing in tempo reale.

## 1. Security Checklist per Deployment (Hardening Gates)
Prima di autorizzare il deploy di un nuovo applicativo, il manifest YAML deve soddisfare i seguenti requisiti di compliance, suddivisi per dominio di sicurezza:

### A. Gestione Identità e Accessi (IAM & RBAC)
* [ ] **RBAC Isolato:** L'applicazione non utilizza il `ServiceAccount` di default, mitigando il rischio di *token hijacking*.
* [ ] **Principio del Least Privilege:** I permessi associati al Pod (`Role` e `RoleBinding`) sono limitati strettamente alle API necessarie.

### B. Sicurezza a Livello Kernel e Compute (Pod Security Context)
* [ ] **Esecuzione Non-Root:** Il parametro `runAsNonRoot` è `true` e l'utente forzato è non privilegiato (es. `runAsUser: 101`).
* [ ] **Privilege Escalation Bloccata:** I parametri `privileged` e `allowPrivilegeEscalation` sono forzati a `false` (prevenzione *Container Breakout*).
* [ ] **Drop delle Capabilities:** È presente il drop esplicito di tutte le capabilities Linux (`drop: ["ALL"]`) per limitare le syscall al kernel.
* [ ] **Isolamento dell'Host:** I flag `hostNetwork`, `hostPID` e `hostIPC` sono disabilitati (`false`).
* [ ] **Infrastruttura Immutabile:** Il filesystem radice è montato in sola lettura (`readOnlyRootFilesystem: true`), delegando le scritture effimere a volumi `emptyDir` isolati (es. `/tmp`).
* [ ] **Prevenzione Starvation:** Sono definiti esplicitamente `requests` e `limits` per mitigare i rischi di saturazione risorse (CPU/RAM).

### C. Sicurezza di Rete (Network Policy)
* [ ] **Isolamento Est-Ovest:** È presente una policy `default-deny-ingress` per impedire il traffico non esplicitamente autorizzato verso il Pod.
* [ ] **Esposizione Sicura:** I servizi utilizzano il tipo `ClusterIP` (esposizione interna) invece di `NodePort`, demandando l'esposizione esterna a un Ingress Controller filtrato.