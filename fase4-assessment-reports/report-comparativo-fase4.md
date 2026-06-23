# Report di Hardening e Mitigazione (Fase 4) - Analisi Tecnica e Troubleshooting

Questo documento descrive le policy di *Hardening* applicate per mitigare le vulnerabilità identificate nella Fase 2, dimostrando l'impatto tecnico delle contromisure sul kernel Linux sottostante e analizzando la risoluzione dei conflitti operativi (*Breaking Changes*).

## 1. Dettaglio Tecnico delle Remediation (Security Context)
Il manifest è stato interamente riscritto applicando il principio del *Least Privilege*:
* **De-escalation dei Privilegi:** I parametri `privileged` e `allowPrivilegeEscalation` sono stati impostati a `false`, impedendo ai processi child di acquisire privilegi superiori rispetto al processo parent (`no_new_privs` flag).
* **Isolamento Syscall e Capabilities:** Applicando `capabilities: drop: ["ALL"]`, il container è stato confinato alle sole System Calls strettamente necessarie, neutralizzando exploit mirati al kernel host.
* **Network Isolation (Default Deny):** L'introduzione di una `NetworkPolicy` di tipo Ingress impone il blocco di tutto il traffico est-ovest (tra pod) non esplicitamente autorizzato, mitigando i movimenti laterali (Lateral Movement) all'interno del cluster.

## 2. Troubleshooting Operativo e Risoluzione Conflitti
L'implementazione rigorosa del Pod Security Standard ha introdotto sfide operative e architetturali, risolte come segue:

### A. Restrizione Binding Porte Privilegiate
* **Conflitto:** Impostando `runAsNonRoot: true` (UID 101), il demone Nginx è andato in stato di `CrashLoopBackOff`. Nel kernel Linux, i processi privi della capability `CAP_NET_BIND_SERVICE` non possono aprire porte di rete inferiori alla 1024 (come la porta HTTP 80 standard).
* **Risoluzione:** Sostituzione con l'immagine `nginx-unprivileged` e spostamento del listener sulla porta non privilegiata `8080`.

### B. Filesystem Immutabile e File PID
* **Conflitto:** L'abilitazione di `readOnlyRootFilesystem: true` previene l'installazione di malware, ma blocca la scrittura dei file di processo (PID) e della cache di Nginx necessari al boot.
* **Risoluzione:** Configurazione di un volume effimero di tipo `emptyDir` (montato in RAM tramite *tmpfs* o su storage temporaneo locale) associato al mount path `/tmp`, permettendo le scritture di sistema senza compromettere l'immutabilità del root filesystem.

### C. Incompatibilità Moduli LSM (Linux Security Modules)
* **Conflitto:** Il profilo `AppArmor` nativo (`RuntimeDefault`) ha impedito l'avvio del Pod nell'ambiente di sviluppo locale.
* **Risoluzione (Analisi):** Il cluster Minikube locale è virtualizzato su un host Windows (tramite WSL2/Hyper-V), privo dei moduli kernel LSM nativi necessari per il binding di AppArmor. L'annotazione è stata commentata per il test locale, confermandone però la necessità assoluta per i nodi worker basati su Linux OS in produzione.

## 3. Validazione Finale e Metriche
Il re-assessment ha confermato il superamento delle criticità:
* **kube-bench:** Da 4 `[FAIL]` critici a **0 `[FAIL]`** nella Sezione 5 (Kubernetes Policies).
* **kubeaudit:** Raggiunto lo score di **0 high-risk vulnerabilities**.
Il workload risulta attualmente resiliente alle principali tecniche di compromissione e container escape.