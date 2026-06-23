# Kubernetes Security Posture Assessment 🛡️

Questo progetto rappresenta il lavoro tecnico svolto durante il mio percorso di tirocinio. L'obiettivo è stato quello di eseguire un assessment completo della postura di sicurezza di un cluster Kubernetes (Minikube), identificare le vulnerabilità di un workload esposto e applicare tecniche di hardening avanzate secondo gli standard del **CIS Benchmark** e il principio del **Least Privilege**.

## Struttura del Progetto

Il lavoro è stato suddiviso in 5 fasi operative, documentate nelle rispettive directory:

* 📁 **Fase 1 (Vulnerability Discovery):** Creazione di un deployment vulnerabile di baseline e mappatura delle misconfigurazioni (esposizione root, container privilegiati, mount insicuri).
* 📁 **Fase 2 (Security Assessment):** Esecuzione di audit infrastrutturali e applicativi utilizzando `kube-bench` e `kubeaudit`. Analisi dei report di vulnerabilità.
* 📁 **Fase 3 (Hardening & Remediation):** Riscrittura dei manifest YAML implementando RBAC restrittivi, Network Policies (Default Deny) e Pod Security Context (Drop capabilities, RunAsNonRoot, ReadOnlyFilesystem).
* 📁 **Fase 4 (Re-Assessment & Troubleshooting):** Validazione post-hardening, misurazione dell'impatto operativo, risoluzione dei conflitti (bypass porte privilegiate, mount temporanei) e ripristino del servizio.
* 📁 **Fase 5 (Standardizzazione):** Produzione di una checklist per gli sviluppatori e di un `golden-manifest.yaml` pronto per essere utilizzato come template aziendale sicuro.

## Tecnologie Utilizzate
* **Kubernetes (Minikube)**
* **kube-bench** (Infrastruttura / CIS Benchmarks)
* **kubeaudit** (Workload Security)
* **Nginx** (Target Application)