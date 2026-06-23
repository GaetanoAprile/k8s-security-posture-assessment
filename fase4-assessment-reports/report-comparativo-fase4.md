# Report Comparativo e Note di Compatibilità (Fase 4)

## Executive Summary
Il presente documento illustra i risultati del re-assessment di sicurezza eseguito dopo l'applicazione delle policy di hardening (Fase 3). L'intervento ha permesso di azzerare le vulnerabilità critiche rilevate nella baseline iniziale, garantendo la totale compliance ai benchmark CIS per la sicurezza dei pod, senza compromettere l'erogazione del servizio applicativo.

---

## 1. Confronto Metriche di Sicurezza (Baseline vs Hardened)
L'efficacia delle remediation è stata misurata ripetendo le scansioni con i tool di auditing:

| Tool di Auditing | Metrica Analizzata | Baseline (Fase 2) | Post-Hardening (Fase 4) | Esito |
| :--- | :--- | :--- | :--- | :--- |
| **kube-bench** | CIS Benchmark (Sez. 5) | 4 `[FAIL]` critici | 0 `[FAIL]` |  Risolto |
| **kubeaudit** | Vulnerabilità Applicative | Molteplici `[ERRO]` | 0 high-risk vulnerabilities | Risolto |

---

## 2. Dettaglio delle Remediation Applicate
Per raggiungere lo score di zero vulnerabilità, il manifest originale è stato corretto applicando le seguenti modifiche architetturali:
* **Mitigazione RBAC:** Rimosso il `ClusterRoleBinding` permissivo. Creato un `ServiceAccount` dedicato con permessi di sola lettura (`get`, `list` sui pod).
* **Chiusura dei Privilegi:** Rimosso il flag `privileged: true` e forzati i parametri `runAsNonRoot: true` e `runAsUser: 101` (utente standard).
* **Isolamento dall'Host:** Impostati a `false` i flag `hostNetwork` e `hostPID` per impedire al pod di accedere alle risorse del nodo fisico.
* **Drop delle Capabilities:** Applicata la direttiva `capabilities: drop: ["ALL"]` per isolare il container dal kernel di base.

---

## 3. Test Funzionali e Verifica del Servizio
I test hanno confermato la corretta erogazione del servizio. L'esposizione insicura tramite `NodePort` è stata sostituita da un servizio interno `ClusterIP`. 
La raggiungibilità è stata verificata instradando il traffico tramite tunnel sicuro:
`kubectl port-forward svc/secure-app-svc 8888:8080`
La connessione a `http://localhost:8888` ha restituito la regolare pagina HTTP 200 di Nginx.

---

## 4. Impatto Operativo e Note di Compatibilità (Troubleshooting)
L'applicazione del *Least Privilege* ha introdotto alcune rotture (breaking changes) previste rispetto al deployment originale, risolte come segue:

1. **Restrizione sulle Porte (Privileged Ports):** Forzando l'utente non-root, Nginx non poteva più aprire la porta standard 80.
   * *Soluzione:* Utilizzo dell'immagine `nginx-unprivileged` e ascolto interno sulla porta `8080`.
2. **Impossibilità di Scrittura (Filesystem Read-Only):** Nginx falliva l'avvio (CrashLoopBackOff) non potendo scrivere i file PID temporanei sul disco blindato in sola lettura.
   * *Soluzione:* Montaggio di un volume effimero `emptyDir` esclusivamente sulla path `/tmp`.
3. **Incompatibilità AppArmor (Ambiente Locale Windows):** Il pod andava in stato `Failed` poiché il kernel host Windows di Minikube non supporta il profilo AppArmor nativo.
   * *Soluzione:* Il profilo è stato temporaneamente commentato per i test locali, ma rimarrà un requisito mandatorio per la produzione.
4. **Isolamento di Rete:** La `NetworkPolicy` di default-deny ha bloccato tutto il traffico in ingresso.
   * *Soluzione:* Comportamento desiderato. In produzione si aggiungeranno regole Ingress esplicite.
