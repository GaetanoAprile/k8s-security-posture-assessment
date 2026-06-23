# Checklist di Sicurezza e Istruzioni Demo (Fase 5)

Questa documentazione fornisce le linee guida per il rilascio di workload sicuri e le istruzioni per eseguire una demo di auditing in tempo reale.

## Checklist per Deployment Sicuri (Hardening)

Prima di effettuare il deploy di una nuova applicazione nel cluster, assicurarsi che il manifest YAML rispetti i seguenti requisiti:

* [ ] **RBAC Personalizzato:** L'applicazione non utilizza il `ServiceAccount` di default.
* [ ] **RBAC Minimo:** I permessi (`Role`/`RoleBinding`) sono limitati al minimo indispensabile (es. sola lettura).
* [ ] **NetworkPolicy:** È presente una policy `default-deny-ingress` per isolare il pod.
* [ ] **Esposizione Sicura:** I servizi non sono esposti direttamente sui nodi (`NodePort`), ma utilizzano `ClusterIP` con un Ingress controller autorizzato.
* [ ] **Esecuzione Non-Root:** Il parametro `runAsNonRoot` è impostato a `true` con un `runAsUser` non privilegiato (es. 101).
* [ ] **Privilegi Disabilitati:** Il parametro `privileged` e `allowPrivilegeEscalation` sono impostati a `false`.
* [ ] **Filesystem Protetto:** Il parametro `readOnlyRootFilesystem` è a `true` (usare `emptyDir` per directory temporanee come `/tmp`).
* [ ] **Capabilities:** È presente il drop esplicito di tutte le capabilities Linux (`drop: ["ALL"]`).
* [ ] **Isolamento Host:** I parametri `hostNetwork`, `hostPID` e `hostIPC` sono impostati a `false`.
* [ ] **Gestione Risorse:** Sono definiti esplicitamente `requests` e `limits` per CPU e Memoria.

---

## Preparazione e Avvio della Demo (Benchmark Output)

Per dimostrare l'efficacia delle configurazioni di sicurezza durante una presentazione, seguire questi step in sequenza:

1. **Ripulire l'ambiente locale:**
   `kubectl delete all --all -n default`
   `kubectl delete networkpolicies --all -n default`

2. **Esecuzione dell'Auditing Applicativo (Kubeaudit):**
   Scansionare il manifest sicuro prima di applicarlo per dimostrare lo score ottimale:
   `kubeaudit all -f golden-manifest.yaml`
   *(Output atteso: `0 high-risk vulnerabilities found`)*

3. **Deploy dell'Applicazione:**
   `kubectl apply -f golden-manifest.yaml`

4. **Verifica dello Stato del Workload:**
   `kubectl get pods`
   *(Output atteso: Pod in stato `Running`)*

5. **Test di Connessione (Bypass Privileged Ports):**
   Avviare il tunnel su una porta non privilegiata per bypassare le restrizioni host:
   `kubectl port-forward svc/app-template-svc 8888:8080`
   *(Dimostrazione: Aprire il browser su `http://localhost:8888` per mostrare il servizio online)*