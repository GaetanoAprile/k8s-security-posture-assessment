# Report Assessment Iniziale (Fase 2) - Analisi delle Vulnerabilità e Postura di Sicurezza


Il presente documento illustra i risultati dell'assessment di sicurezza condotto sul cluster locale (Minikube) e sull'applicazione vulnerabile di test (*vulnerable-demo.yaml*). L'obiettivo di questa fase è mappare la superficie di attacco e valutare la deviazione dell'infrastruttura rispetto alle best practice internazionali.

L'analisi è stata condotta adottando un approccio a doppio livello:
1. **Auditing Infrastrutturale:** Utilizzo di **kube-bench** per validare la configurazione del nodo e del control plane rispetto agli standard del *CIS (Center for Internet Security) Kubernetes Benchmark*.
2. **Static Application Security Testing (SAST):** Utilizzo di **kubeaudit** per l'ispezione statica del codice YAML, al fine di individuare misconfigurazioni legate al *Pod Security Context* e all'isolamento dei workload.

---

## 1. Risultati Assessment Infrastrutturale (kube-bench)
L'esecuzione di kube-bench ha generato un totale di 14 check falliti (`[FAIL]`). Di questi, la maggior parte è imputabile all'architettura di default del cluster locale Minikube (es. assenza di audit logging sul Control Plane e permessi sui file di configurazione dell'API Server), considerata fuori perimetro per il presente progetto.

Focalizzandoci invece sulla **Sezione 5 (Kubernetes Policies)**, che impatta direttamente i workload applicativi, sono emersi **4 Finding critici** direttamente causati dal nostro manifest:

* **[FAIL] CIS 5.1.1 - Ensure that the cluster-admin role is only used where required:** Rilevato l'uso di un `ClusterRoleBinding` eccessivamente permissivo associato al `ServiceAccount` di default. Questa configurazione viola il principio del *Least Privilege*, garantendo al pod poteri assoluti sull'intero cluster.
* **[FAIL] CIS 5.2.2 - Minimize the admission of privileged containers:** Rilevato l'uso del flag `privileged: true`. Questo disabilita le restrizioni cgroups e i namespaces di Linux, rendendo il container di fatto un processo root sull'host fisico.
* **[FAIL] CIS 5.2.3 / 5.2.5 - Minimize sharing host process / network namespaces:** Rilevato l'uso anomalo di `hostPID: true` e `hostNetwork: true`. Il pod è in grado di intercettare il traffico di rete del nodo e visualizzare i processi (PID) in esecuzione sulle altre partizioni del sistema operativo.
* **[FAIL] CIS 5.2.7 - Minimize the admission of root containers:** Il container è configurato per l'esecuzione come utente root (`runAsUser: 0`), esponendo il sistema a rischi di *Privilege Escalation*.

---

## 2. Risultati Assessment Applicativo (kubeaudit)
L'analisi del manifest YAML tramite kubeaudit ha restituito molteplici vulnerabilità ad alto rischio (identificate come `[ERRO]`), confermando i finding di kube-bench e aggiungendo ulteriore granularità a livello di kernel Linux:

* **CapabilityOrSecurityContextMissing:** Mancanza di un drop esplicito delle capabilities del kernel Linux (`drop: ["ALL"]`). Il container mantiene capabilities pericolose di default (es. `CAP_NET_RAW` e `CAP_CHOWN`).
* **ReadOnlyRootFilesystemNil:** Il filesystem del container non è limitato in sola lettura (`readOnlyRootFilesystem: false`). Questo permette a un potenziale attaccante di scaricare ed eseguire payload malevoli (malware/ransomware) direttamente all'interno del pod.
* **AppArmorAnnotationMissing & SeccompProfileMissing:** Assenza di profili di sicurezza LSM (Linux Security Modules) per limitare e filtrare le chiamate di sistema (syscall) dirette dal container al kernel dell'host.
* **LimitsNotSet [WARN]:** Assenza di resource quota (`requests` e `limits` per CPU/RAM), esponendo il nodo worker a colli di bottiglia, resource starvation o attacchi DoS (Denial of Service) volontari.

---

## 3. Analisi del Rischio e Vettori di Attacco
L'insieme delle misconfigurazioni rilevate espone l'architettura a scenari di compromissione totale (*Cluster Takeover*). Nello specifico, la combinazione di `privileged: true`, esecuzione come Root e condivisione dei namespace host rende banale l'esecuzione di un attacco di **Container Breakout** (o Container Escape). 
Un attaccante in grado di sfruttare una vulnerabilità nell'applicativo esposto potrebbe evadere dal perimetro del container e ottenere l'accesso shell diretto al nodo worker sottostante.

---

## 4. Mappatura e Identificazione dei "Quick Wins" (Piano di Remediation)
Per mitigare rapidamente i rischi maggiori con il minimo sforzo architetturale, gli interventi di hardening (Fase 3) si concentreranno sulle seguenti priorità strategiche:

1. **Quick Win 1 (RBAC & Identity):** Rimozione del `ClusterRoleBinding` globale e creazione di un `ServiceAccount` dedicato all'applicativo, associato a un `Role` con permessi limitati esclusivamente alle risorse strettamente necessarie.
2. **Quick Win 2 (Pod Security & Isolation):** Inserimento di un blocco `securityContext` stringente per forzare l'esecuzione non-root (`runAsNonRoot: true`), rimuovere i privilegi (`privileged: false`), bloccare l'escalation dei processi (`allowPrivilegeEscalation: false`) e montare il filesystem in sola lettura.
3. **Quick Win 3 (Network Security):** Sostituzione del servizio `NodePort` con un servizio `ClusterIP` interno per chiudere l'accesso diretto e non filtrato alle porte del nodo fisico, predisponendo l'architettura per una futura gestione tramite Ingress Controller.