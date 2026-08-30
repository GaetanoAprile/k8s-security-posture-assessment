# Report di Hardening e Mitigazione (Fase 4) — Analisi Tecnica e Troubleshooting

> **Nota di revisione:** questa versione estende il report originale con la validazione dell'enforcement runtime delle NetworkPolicy (§2.D, §3), un secondo conflitto emerso durante il redo (§2.E), e la correzione della metrica kube-bench in §3, risultata non significativa per il confronto prima/dopo.

Questo documento descrive le policy di *Hardening* applicate per mitigare le vulnerabilità identificate nella Fase 2, dimostrando l'impatto tecnico delle contromisure sul kernel Linux sottostante e analizzando la risoluzione dei conflitti operativi (*Breaking Changes*), inclusi quelli emersi durante la validazione dell'enforcement a livello di rete.

---

## 1. Dettaglio Tecnico delle Remediation (Security Context)

Il manifest è stato interamente riscritto applicando il principio del *Least Privilege*:

* **De-escalation dei Privilegi:** i parametri `privileged` e `allowPrivilegeEscalation` sono stati impostati a `false`, impedendo ai processi child di acquisire privilegi superiori rispetto al processo parent (flag `no_new_privs`).
* **Isolamento Syscall e Capabilities:** applicando `capabilities: drop: ["ALL"]`, il container è stato confinato alle sole system call strettamente necessarie, neutralizzando exploit mirati al kernel host.
* **Network Isolation (Default Deny):** sono state introdotte due NetworkPolicy distinte, applicate allo stesso `podSelector`:
  * una di tipo `Ingress`, che blocca tutto il traffico in entrata non esplicitamente autorizzato, con un'allow-rule verso i client recanti la label `role: authorized-client`;
  * una di tipo `Egress`, che blocca tutto il traffico in uscita ad eccezione della risoluzione DNS verso `kube-system` (porta 53, UDP/TCP), necessaria al funzionamento di base del Pod.

  *(La prima stesura del manifest includeva solo la componente Ingress. L'assenza della componente Egress lasciava aperto un canale di comunicazione in uscita non filtrato, potenzialmente sfruttabile per l'esfiltrazione di dati o per un callback verso un'infrastruttura esterna — lo stesso vettore "Abuso di risorse" descritto in §1.1. La policy di Egress è stata aggiunta in fase di validazione.)*

---

## 2. Troubleshooting Operativo e Risoluzione Conflitti

L'implementazione rigorosa del Pod Security Standard ha introdotto sfide operative e architetturali, risolte come segue:

### A. Restrizione Binding Porte Privilegiate
* **Conflitto:** impostando `runAsNonRoot: true` (UID 101), il processo Nginx è andato in stato di `CrashLoopBackOff`. Nel kernel Linux, i processi privi della capability `CAP_NET_BIND_SERVICE` non possono aprire porte di rete inferiori alla 1024 (come la porta HTTP 80 standard).
* **Risoluzione:** sostituzione con l'immagine `nginx-unprivileged` e spostamento del listener sulla porta non privilegiata `8080`.

### B. Filesystem Immutabile e File PID
* **Conflitto:** l'abilitazione di `readOnlyRootFilesystem: true` previene l'installazione di malware, ma blocca la scrittura dei file di processo (PID) e della cache di Nginx necessari al boot.
* **Risoluzione:** configurazione di un volume effimero di tipo `emptyDir` montato su `/tmp`, permettendo le scritture di sistema senza compromettere l'immutabilità del root filesystem.

### C. Incompatibilità Moduli LSM (Linux Security Modules)
* **Conflitto:** il profilo AppArmor nativo (`RuntimeDefault`) ha impedito l'avvio del Pod nell'ambiente di sviluppo locale.
* **Risoluzione (Analisi):** il cluster Minikube locale è virtualizzato su un host Windows tramite WSL2, privo dei moduli kernel LSM nativi necessari per il binding di AppArmor. Il campo `securityContext.appArmorProfile` è stato commentato per il test locale, confermandone però la necessità per i nodi worker basati su Linux OS in produzione.

### D. Enforcement delle NetworkPolicy e Scelta del CNI
* **Conflitto:** le NetworkPolicy applicate a un cluster Minikube con installazione di default (`minikube start`, driver Docker) non producono alcun effetto reale sul traffico di rete. Il CNI predefinito di Minikube, Kindnet, non implementa il supporto alle NetworkPolicy per design — l'oggetto rimane un artefatto valido nell'API di Kubernetes, ma inerte a runtime. Poiché kube-bench e kubeaudit eseguono unicamente analisi statica del manifest, nessuno dei due strumenti è in grado di rilevare questa limitazione.
* **Risoluzione:** il cluster è stato ricreato specificando un CNI compatibile (`minikube start --cni=calico`), e l'intera sequenza di assessment (Fase 1 → Fase 4) è stata ripetuta sul cluster ricreato, per garantire la comparabilità dei risultati a parità di infrastruttura sottostante. La verifica funzionale dell'enforcement è descritta in §3.

### E. Annotazione AppArmor Legacy e Rifiuto del Pod
* **Conflitto:** durante la validazione sul cluster ricreato, il Pod del golden manifest veniva sistematicamente rifiutato dal kubelet (`Status: Failed`, `Reason: AppArmor`, `Message: Cannot enforce AppArmor: AppArmor is not enabled on the host`). La causa era la presenza, nei metadata del Pod template, dell'annotazione legacy `container.apparmor.security.beta.kubernetes.io/<container>`, rimasta attiva in parallelo al campo moderno `securityContext.appArmorProfile` (quest'ultimo correttamente commentato, come da §2.C).
* **Risoluzione:** rimozione dell'annotazione legacy, mantenendo solo il meccanismo moderno. Il conflitto conferma, tramite un meccanismo diverso da quello di §2.C, lo stesso limite ambientale: il nodo utilizzato non dispone del supporto LSM necessario per l'enforcement di AppArmor, indipendentemente da quale dei due meccanismi Kubernetes venga usato per dichiararlo.

---

## 3. Validazione Finale e Metriche

Il re-assessment è stato condotto su tre livelli indipendenti, a copertura sia della conformità statica sia dell'enforcement runtime effettivo.

**Conformità statica del workload (kubeaudit).** Applicato al golden manifest, kubeaudit riporta `0 high-risk vulnerabilities` — a fronte degli 11 finding rilevati da kubeaudit sul baseline in Fase 2 (10 `[error]` + 1 `[warning]`), tutti risolti. A questi si aggiunge il `ClusterRoleBinding` verso `cluster-admin` confermato in Fase 2 §2 tramite ispezione manuale (non rilevabile da kubeaudit): anche questo risolto nel golden manifest, sostituito da un `Role` con permessi minimi (si veda il report di Fase 2 corretto per il dettaglio completo, 12 finding totali).

**Conformità infrastrutturale (kube-bench, Sezioni 1–4).** Invariata rispetto al baseline (14 `[FAIL]`, riconducibili alla configurazione di default di Minikube) — atteso, poiché queste sezioni non dipendono dal manifest applicativo.

> **Correzione rispetto alla versione originale.** La precedente stesura di questo report riportava *"kube-bench: da 4 [FAIL] critici a 0 [FAIL] nella Sezione 5"*. L'affermazione va corretta: i controlli di Sezione 5 rilevanti (5.1.1, 5.2.2, 5.2.3/5.2.5, 5.2.7) sono di tipo *"(Manual)"* e restituiscono sempre `[WARN]` in kube-bench, sia sul baseline sia sul golden manifest — non costituiscono quindi un indicatore quantitativo di miglioramento utilizzabile. Il confronto quantitativo corretto sul workload è quello su kubeaudit, riportato sopra.

**Enforcement runtime (test di connettività).** Per verificare che le NetworkPolicy non fossero solo dichiarate ma effettivamente applicate dal CNI, è stato condotto un test funzionale con due pod client `busybox`:

```powershell
# Client non autorizzato
kubectl run unauthorized-client --image=busybox --restart=Never -- wget -qO- --timeout=2 http://app-template-svc:8080

# Client autorizzato (label role: authorized-client)
kubectl run authorized-client --image=busybox --restart=Never --labels="role=authorized-client" -- wget -qO- --timeout=2 http://app-template-svc:8080
```

**Risultato:** il client non autorizzato termina in stato `Error`, con log `wget: download timed out` — il pacchetto viene scartato senza risposta, comportamento tipico di un blocco effettivo a livello di rete. Il client autorizzato termina in stato `Completed`, ricevendo correttamente la risposta HTTP del servizio. La differenza qualitativa tra i due esiti (timeout vs. risposta) costituisce la prova sperimentale che l'enforcement delle NetworkPolicy, tramite CNI Calico, è realmente attivo — non solo dichiarato nel manifest.

Il workload risulta resiliente alle principali tecniche di compromissione e container escape testate, con validazione sia a livello di configurazione sia a livello di comportamento di rete effettivo.