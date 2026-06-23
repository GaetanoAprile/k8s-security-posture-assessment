# Report Assessment Iniziale (Fase 2)

Il presente documento illustra i risultati dell'assessment di sicurezza condotto sul cluster locale (Minikube) e sull'applicazione vulnerabile di test. L'analisi è stata effettuata utilizzando due strumenti complementari: **kube-bench** per l'infrastruttura/policy e **kubeaudit** per i manifest applicativi.

## 1. Risultati Assessment Infrastrutturale (kube-bench)
L'esecuzione di kube-bench ha generato un totale di 14 check falliti ([FAIL]). Di questi, la maggior parte è imputabile alla configurazione di default del cluster locale Minikube (es. assenza di audit logging sul Control Plane), considerata fuori perimetro per questo progetto.

Focalizzandoci invece sulla **Sezione 5 (Kubernetes Policies)**, che impatta direttamente i workload applicativi, sono emersi i seguenti 4 *Finding* critici direttamente legati al nostro file YAML:

* **[FAIL] CIS 5.1.1** - *Ensure that the cluster-admin role is only used where required*: Rilevato l'uso di un RoleBinding eccessivamente permissivo associato al ServiceAccount di default.
* **[FAIL] CIS 5.2.2** - *Minimize the admission of privileged containers*: Rilevato l'uso del flag `privileged: true`.
* **[FAIL] CIS 5.2.3 / 5.2.5** - *Minimize sharing host process / network namespaces*: Rilevato l'uso anomalo di `hostPID: true` e `hostNetwork: true`.
* **[FAIL] CIS 5.2.7** - *Minimize the admission of root containers*: Il container è configurato per l'esecuzione come utente root (`runAsUser: 0`).

## 2. Risultati Assessment Applicativo (kubeaudit)
L'analisi del manifest YAML (`vulnerable-demo.yaml`) tramite kubeaudit ha restituito molteplici vulnerabilità ad alto rischio (identificate come `[ERRO]`), confermando e dettagliando i finding di kube-bench:

* **CapabilityOrSecurityContextMissing:** Mancanza di un drop esplicito delle capabilities del kernel Linux.
* **ReadOnlyRootFilesystemNil:** Il filesystem del container non è limitato in sola lettura, permettendo potenziali scritture malevole.
* **AppArmorAnnotationMissing & SeccompProfileMissing:** Assenza di profili di sicurezza per limitare le chiamate di sistema del container.
* **LimitsNotSet [WARN]:** Assenza di resource quota (CPU/RAM), esponendo il cluster a colli di bottiglia o attacchi DoS.

## 3. Mappatura e Identificazione dei "Quick Wins"
Per mitigare rapidamente i rischi maggiori con il minimo sforzo architetturale (Quick Wins), gli interventi di remediation (Fase 3) si concentreranno sulle seguenti priorità:

1. **Quick Win 1 (RBAC):** Rimozione del `ClusterRoleBinding` e creazione di un `ServiceAccount` dedicato privo di permessi amministrativi.
2. **Quick Win 2 (Pod Security):** Inserimento di un blocco `securityContext` stringente per forzare l'esecuzione non-root (`runAsNonRoot: true`), rimuovere i privilegi (`privileged: false`) e bloccare l'escalation (`allowPrivilegeEscalation: false`).
3. **Quick Win 3 (Network):** Sostituzione del servizio `NodePort` con un servizio `ClusterIP` per chiudere l'accesso diretto alle porte del nodo fisico.