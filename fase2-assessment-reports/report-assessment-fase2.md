# Report Assessment Iniziale (Fase 2) — Analisi delle Vulnerabilità e Postura di Sicurezza

> **Nota di revisione:** questa versione corregge l'attribuzione di quattro finding, originariamente indicati come `[FAIL]` di kube-bench. La verifica incrociata condotta in fase di validazione (redo con CNI Calico) ha chiarito che kube-bench non emette mai un verdetto `[FAIL]` automatico sui controlli della Sezione 5 (sono di tipo "Manual"); il verdetto puntuale su questi controlli proviene da kubeaudit. Le vulnerabilità restano identiche e reali — cambia solo quale strumento le ha effettivamente rilevate.

Il presente documento illustra i risultati dell'assessment di sicurezza condotto sul cluster locale (Minikube) e sull'applicazione vulnerabile di test (*vulnerable-demo.yaml*). L'obiettivo di questa fase è mappare la superficie di attacco e valutare la deviazione dell'infrastruttura rispetto alle best practice internazionali.

L'analisi è stata condotta adottando un approccio a doppio livello:
1. **Auditing Infrastrutturale:** utilizzo di **kube-bench** per validare la configurazione del nodo e del control plane rispetto agli standard del *CIS (Center for Internet Security) Kubernetes Benchmark*.
2. **Static Application Security Testing (SAST):** utilizzo di **kubeaudit** per l'ispezione statica del manifest YAML, al fine di individuare misconfigurazioni legate al *Pod Security Context* e all'isolamento dei workload.

---

## 1. Risultati Assessment Infrastrutturale (kube-bench)

L'esecuzione di kube-bench ha rilevato un totale di **14 check falliti** (`[FAIL]`), interamente concentrati nelle Sezioni 1–4 del benchmark (Control Plane Security Configuration, Etcd Node Configuration, Control Plane Configuration, Worker Node Security Configuration). Questi fallimenti sono riconducibili alla configurazione di default del cluster locale Minikube (es. permessi dei file di configurazione dell'API Server, assenza di audit logging, permessi della directory etcd) — un'area considerata fuori perimetro per il presente progetto, che si concentra sulla sicurezza del workload applicativo piuttosto che sull'hardening del Control Plane.

**Nota metodologica sulla Sezione 5 (Kubernetes Policies).** La quasi totalità dei controlli di questa sezione è implementata da kube-bench come check di tipo *"(Manual)"*: per questi controlli lo strumento non emette mai un verdetto `[FAIL]` automatico, indipendentemente dallo stato reale del cluster, ma restituisce sempre `[WARN]`, segnalando la necessità di una verifica manuale. Di conseguenza, in questo report kube-bench viene impiegato principalmente come riferimento alla numerazione ufficiale del CIS Kubernetes Benchmark; il verdetto di conformità puntuale sul workload — l'informazione realmente actionable ai fini dell'assessment — viene fornito da kubeaudit, illustrato nella sezione seguente.

---

## 2. Risultati Assessment Applicativo (kubeaudit)

L'analisi statica del manifest tramite kubeaudit ha restituito **11 finding**, di cui 10 a severità `[error]` e 1 a severità `[warning]`:

| Auditor kubeaudit | Severità | Rif. CIS | Descrizione |
|---|---|---|---|
| `PrivilegedTrue` | error | 5.2.2 | Container eseguito in modalità `privileged: true` |
| `NamespaceHostPIDTrue` | error | 5.2.3 | Pod configurato con `hostPID: true` |
| `NamespaceHostNetworkTrue` | error | 5.2.5 | Pod configurato con `hostNetwork: true` |
| `RunAsUserCSCRoot` | error | 5.2.7 | Container eseguito con `runAsUser: 0` (root) |
| `AllowPrivilegeEscalationNil` | error | 5.2.6 | `allowPrivilegeEscalation` non impostato esplicitamente a `false` |
| `CapabilityOrSecurityContextMissing` | error | 5.2.9 | Nessun drop esplicito delle Linux capabilities |
| `ReadOnlyRootFilesystemNil` | error | — | `readOnlyRootFilesystem` non impostato a `true` |
| `SeccompProfileMissing` | error | 5.6.2 | Profilo Seccomp assente nel Pod SecurityContext |
| `AppArmorAnnotationMissing` | error | — | Annotazione AppArmor assente |
| `AutomountServiceAccountTokenTrueAndDefaultSA` | error | 5.1.5 / 5.1.6 | Token del ServiceAccount di default montato nel Pod |
| `LimitsNotSet` | warning | — | Resource limits (CPU/memoria) non impostati |

Le voci prive di un riferimento CIS diretto (`ReadOnlyRootFilesystemNil`, `AppArmorAnnotationMissing`, `LimitsNotSet`) non corrispondono a un controllo numerato esplicito della Sezione 5, ma rientrano nelle best practice di hardening documentate rispettivamente dal Pod Security Context (§2.6.2) e dalla NSA/CISA Kubernetes Hardening Guide.

> **Controllo CIS 5.1.1 (cluster-admin) — confermato tramite verifica manuale.** kubeaudit non dispone di un auditor dedicato all'analisi di ClusterRoleBinding: la conferma di questo controllo, correttamente segnalato da kube-bench come `[WARN]` (Manual) e non verificabile in automatico, proviene dall'ispezione diretta del manifest. `vulnerable-demo.yaml` contiene un `ClusterRoleBinding` (`bad-rbac-binding`) che associa la `ClusterRole` `cluster-admin` al `ServiceAccount` `default` del namespace `default` — lo stesso ServiceAccount ereditato implicitamente da qualsiasi Pod del namespace che non ne specifichi uno proprio. Il finding aggrava inoltre `AutomountServiceAccountTokenTrueAndDefaultSA` (Sezione 2): il token del ServiceAccount di default viene montato automaticamente in ogni Pod e, a causa di questo binding, quel token garantisce permessi di amministratore sull'intero cluster. Un attaccante che compromettesse un qualsiasi Pod del namespace `default` privo di configurazione esplicita otterrebbe quindi, tramite il token montato, pieno controllo dell'API di Kubernetes.

---

## 3. Analisi del Rischio e Vettori di Attacco

L'insieme delle misconfigurazioni rilevate espone l'architettura a scenari di compromissione totale (*Cluster Takeover*). Nello specifico, la combinazione di `privileged: true`, esecuzione come Root e condivisione dei namespace host rende banale l'esecuzione di un attacco di **Container Breakout** (o Container Escape).

Un attaccante in grado di sfruttare una vulnerabilità nell'applicativo esposto potrebbe evadere dal perimetro del container e ottenere l'accesso shell diretto al nodo worker sottostante.

---

## 4. Mappatura e Identificazione dei "Quick Win" (Piano di Remediation)

Per mitigare rapidamente i rischi maggiori con il minimo sforzo architetturale, gli interventi di hardening (Fase 3) si sono concentrati sulle seguenti priorità strategiche:

1. **Quick Win 1 (RBAC & Identity):** rimozione del `ClusterRoleBinding` `bad-rbac-binding` (cluster-admin → ServiceAccount default, confermato in Sezione 2) e creazione di un `ServiceAccount` dedicato all'applicativo, associato a un `Role` con permessi limitati esclusivamente alle risorse strettamente necessarie. Risolve inoltre il finding `AutomountServiceAccountTokenTrueAndDefaultSA`.
2. **Quick Win 2 (Pod Security & Isolation):** inserimento di un blocco `securityContext` stringente per forzare l'esecuzione non-root, rimuovere i privilegi, bloccare l'escalation dei processi e montare il filesystem in sola lettura. Risolve i finding `PrivilegedTrue`, `RunAsUserCSCRoot`, `AllowPrivilegeEscalationNil`, `ReadOnlyRootFilesystemNil`, `CapabilityOrSecurityContextMissing`, `SeccompProfileMissing`, `NamespaceHostPIDTrue`, `NamespaceHostNetworkTrue`.
3. **Quick Win 3 (Network Security):** sostituzione del servizio `NodePort` con un servizio `ClusterIP` interno, predisponendo l'architettura per una gestione tramite Ingress Controller e NetworkPolicy dedicate (dettagliate in Fase 3/4).

`LimitsNotSet` non rientra in nessuno dei tre Quick Win originali, ma è stato comunque risolto nel golden manifest finale tramite l'impostazione di `resources.requests`/`resources.limits`.