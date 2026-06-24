# formazione_sou_k8s

Repository di formazione per il deploy di applicazioni containerizzate su Kubernetes tramite Helm, con pipeline di consegna continua orchestrata da Jenkins.

Il caso d'uso documentato in questo repository è il deploy dell'applicazione `flask-app-example`, la cui immagine Docker è prodotta dalla pipeline `flask-app-example-build` e pubblicata su Docker Hub.

## Indice

- [Struttura del repository](#struttura-del-repository)
- [Architettura della pipeline](#architettura-della-pipeline)
- [Il Helm Chart: flask-app-example](#il-helm-chart-flask-app-example)
- [Pipeline CI/CD (Jenkins)](#pipeline-cicd-jenkins)
- [Prerequisiti infrastrutturali](#prerequisiti-infrastrutturali)
- [Utilizzo manuale (senza Jenkins)](#utilizzo-manuale-senza-jenkins)
- [Convenzioni di versioning](#convenzioni-di-versioning)
- [Risoluzione dei problemi comuni](#risoluzione-dei-problemi-comuni)

---

## Struttura del repository

```
formazione_sou_k8s/
└── charts/
    └── flask-app-example/
        ├── Chart.yaml          Metadati del chart (nome, versione)
        ├── values.yaml         Valori di default (immagine, porte, service, ingress)
        ├── .helmignore         File esclusi dal packaging del chart
        ├── Jenkinsfile         Pipeline dichiarativa di deploy
        ├── README.md           Documentazione specifica del chart
        └── templates/
            ├── _helpers.tpl    Funzioni di template riutilizzabili (naming, label)
            ├── deployment.yaml Deployment Kubernetes dell'applicazione
            ├── service.yaml    Service Kubernetes per l'esposizione interna
            └── ingress.yaml    Ingress opzionale per l'esposizione esterna
```

Il `Jenkinsfile` è collocato volutamente dentro la cartella del chart che gestisce, non alla radice del repository. Il job Jenkins corrispondente è configurato con questo path specifico nello SCM.

---

## Architettura della pipeline

Il flusso di consegna, dal codice alla risorsa in esecuzione nel cluster, segue questo percorso:

```
┌───────────────────────┐
│  GitHub                │
│  formazione_sou_k8s    │
└───────────┬────────────┘
            │ checkout scm
            ▼
┌───────────────────────┐
│  Jenkins                │
│  Pipeline dichiarativa  │
│  (trigger manuale)      │
│  parametro: IMAGE_TAG   │
└───────────┬────────────┘
            │ helm upgrade --install
            │ --set image.tag=<TAG>
            ▼
┌───────────────────────┐
│  kubectl proxy          │
│  (tunnel di rete)       │
│  192.168.56.1:8001      │
└───────────┬────────────┘
            │
            ▼
┌───────────────────────┐
│  Kubernetes (Minikube)  │
│  namespace:             │
│  formazione-sou         │
│                          │
│  Deployment + Service   │
│  (+ Ingress opzionale)  │
└───────────┬────────────┘
            │
            ▼
┌───────────────────────┐
│  Pod in esecuzione      │
│  immagine Docker Hub:   │
│  <org>/flask-app-       │
│  example:<TAG>          │
└───────────────────────┘
```

Il tunnel di rete e i permessi RBAC che rendono possibile il collegamento Jenkins → Kubernetes sono dettagliati nella sezione [Prerequisiti infrastrutturali](#prerequisiti-infrastrutturali).

---

## Il Helm Chart: flask-app-example

Il chart pacchettizza il deploy dell'applicazione Flask come Deployment, Service e, opzionalmente, Ingress. La documentazione completa, con la spiegazione di ogni parametro, è in [`charts/flask-app-example/README.md`](charts/flask-app-example/README.md).

Le decisioni di progettazione principali:

| Aspetto | Scelta | Motivazione |
|---|---|---|
| Tag immagine | Parametro esplicito (`image.tag`), nessun fallback silenzioso | Garantisce deploy riproducibili: nessuna installazione "accidentale" di una versione non intenzionale |
| Registry | Docker Hub (`<dockerhub-org>/flask-app-example`) | Coerente con la pipeline `flask-app-example-build` |
| Porta applicativa | `containerPort` separato da `service.port` | L'app Flask e l'esposizione di rete possono evolvere indipendentemente |
| Ingress | Disattivato di default (`ingress.enabled: false`) | Richiede precondizioni esterne (Ingress Controller); resta opt-in |
| Service type | `ClusterIP` | L'esposizione esterna è delegata all'Ingress, non al Service |

Il parametro su cui interviene la pipeline ad ogni deploy è `image.tag`:

```bash
helm upgrade --install flask-app-example ./charts/flask-app-example \
  --namespace formazione-sou \
  --set image.tag=<TAG>
```

---

## Pipeline CI/CD (Jenkins)

### Trigger

La pipeline è **manuale**: viene avviata dall'interfaccia Jenkins specificando il parametro `IMAGE_TAG`, che deve corrispondere a un tag effettivamente pubblicato su Docker Hub per l'immagine `flask-app-example`.

### Parametro

| Nome | Tipo | Default | Descrizione |
|---|---|---|---|
| `IMAGE_TAG` | string | `latest` | Tag dell'immagine Docker Hub da rilasciare, prodotta dalla pipeline `flask-app-example-build` |

### Stage della pipeline

| Stage | Azione |
|---|---|
| Checkout chart da GitHub | `checkout scm`: recupera il repository (incluso il chart) nella workspace dell'agent |
| Deploy con Helm | Recupera il kubeconfig dedicato dalle Credentials di Jenkins ed esegue `helm upgrade --install` sul namespace `formazione-sou` |

### Note implementative

- Si usa `helm upgrade --install`, non `helm install`: rende la pipeline idempotente, eseguibile più volte senza errori di "release già esistente".
- Il kubeconfig è recuperato tramite `withCredentials([file(credentialsId: 'kubeconfig-formazione-sou', variable: 'KUBECONFIG')])`: il file resta cifrato a riposo nel Credentials Store di Jenkins e viene scritto su disco solo temporaneamente, per la durata dello stage.
- I parametri della pipeline sono referenziati nella shell come variabili d'ambiente (`$IMAGE_TAG`), non tramite interpolazione Groovy (`${params.IMAGE_TAG}`) all'interno di stringhe a virgolette singole, per evitare ambiguità tra sintassi Groovy e sintassi shell.

---

## Prerequisiti infrastrutturali

L'ambiente di sviluppo locale è composto da più livelli di virtualizzazione che non comunicano tra loro per default. Questa sezione documenta come sono stati collegati.

### Topologia

```
┌───────────────────────────────── Mac (host) ──────────────────────────────────┐
│                                                                                  │
│   Docker Desktop                                                                │
│   └── Minikube (cluster Kubernetes, driver Docker)                             │
│       API Server: 192.168.49.2 — raggiungibile solo dal Mac                    │
│                                                                                  │
│   kubectl proxy --address=192.168.56.1 --accept-hosts='^.*$' --port=8001        │
│   (bridge tra la rete interna di Docker Desktop e la rete privata Vagrant)      │
│                                                                                  │
│   ┌────────────────────── VM Vagrant (provider: VirtualBox) ──────────────────┐ │
│   │   rete privata condivisa: 192.168.56.10                                   │ │
│   │                                                                            │ │
│   │   Container Podman — Jenkins (porta 8080 forwarded sul Mac)               │ │
│   │   Container Podman — Agent persistente (git, helm 3.15.4)                 │ │
│   │                                                                            │ │
│   └─────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1. Tunnel di rete

Il cluster Minikube, girando dentro Docker Desktop, non è raggiungibile dalla VM Vagrant. Il collegamento è realizzato con `kubectl proxy`, in ascolto sull'interfaccia di rete privata condivisa con la VM (non solo su `localhost`):

```bash
kubectl proxy --address='192.168.56.1' --accept-hosts='^.*$' --port=8001
```

Questo comando deve restare attivo sul Mac perché la pipeline possa raggiungere il cluster. `--accept-hosts='^.*$'` rimuove il filtro di default del proxy, che altrimenti accetterebbe solo richieste apparentemente locali.

### 2. Namespace e permessi (RBAC)

Jenkins opera con un'identità dedicata, non con le credenziali amministrative personali, limitata al solo namespace `formazione-sou`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-deployer
  namespace: formazione-sou
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer-admin
  namespace: formazione-sou
subjects:
  - kind: ServiceAccount
    name: jenkins-deployer
    namespace: formazione-sou
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

L'uso di un `RoleBinding` (e non di un `ClusterRoleBinding`) è ciò che limita l'effetto del `ClusterRole admin` al namespace `formazione-sou`, impedendo l'accesso ad altri namespace del cluster.

Il token di autenticazione per il ServiceAccount è generato con:

```bash
kubectl create token jenkins-deployer -n formazione-sou --duration=87600h
```

### 3. Credenziale Jenkins

Il token, insieme all'indirizzo del tunnel, è incapsulato in un kubeconfig dedicato e caricato in Jenkins come Credential di tipo **Secret file**:

| Campo | Valore |
|---|---|
| ID Credential | `kubeconfig-formazione-sou` |
| Tipo | Secret file |
| Scope | Global |

### 4. Requisiti dell'agent Jenkins

L'agent persistente (container Podman, immagine basata su Rocky Linux) richiede `git` e `helm` installati. La versione di Helm è fissata esplicitamente nel Dockerfile dell'agent, per garantire build riproducibili:

```dockerfile
RUN dnf install -y git

ARG HELM_VERSION=3.15.4
RUN curl -fsSL -o /tmp/helm.tar.gz https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz \
    && tar -zxvf /tmp/helm.tar.gz -C /tmp \
    && mv /tmp/linux-amd64/helm /usr/local/bin/helm \
    && rm -rf /tmp/helm.tar.gz /tmp/linux-amd64
```

---

## Utilizzo manuale (senza Jenkins)

Per validare o installare il chart direttamente da terminale, senza passare dalla pipeline:

```bash
# Controllo strutturale del chart
helm lint ./charts/flask-app-example

# Rendering dei manifest, senza installare nulla
helm template flask-app-example ./charts/flask-app-example --set image.tag=<TAG>

# Installazione o aggiornamento nel cluster
helm upgrade --install flask-app-example ./charts/flask-app-example \
  --namespace formazione-sou \
  --create-namespace \
  --set image.tag=<TAG>

# Rollback alla revisione precedente, in caso di problemi
helm rollback flask-app-example
```

---

## Convenzioni di versioning

Il repository distingue tre numeri di versione indipendenti, che è importante non confondere:

| Campo | Dove si trova | Cosa rappresenta | Quando cambia |
|---|---|---|---|
| `version` | `Chart.yaml` | Versione del chart stesso (la struttura dei template) | Quando si modifica un template o un parametro in `values.yaml` |
| `appVersion` | `Chart.yaml` | Segnaposto informativo, non vincolante | Raramente; non è collegato al tag effettivamente deployato |
| `image.tag` | Parametro a runtime (`--set` o pipeline) | Versione reale dell'immagine applicativa deployata | Ad ogni deploy, deve corrispondere a un tag esistente su Docker Hub |

---

## Risoluzione dei problemi comuni

| Sintomo | Causa probabile | Verifica | Risoluzione |
|---|---|---|---|
| `nil pointer evaluating interface {}.campo` durante `helm template` | Un template referenzia un campo che non esiste (più) in `values.yaml` | Controllare il campo indicato nell'errore e la sua presenza in `values.yaml` | Aggiungere il campo mancante, oppure rimuovere/adattare la porzione di template che lo referenzia |
| Un valore atteso non compare nel manifest renderizzato (es. porta diversa da quella configurata) | Il template referenzia un campo `.Values.x` diverso da quello realmente impostato | `helm template . --debug` per ispezionare i valori effettivamente caricati; controllare a cosa punta il placeholder `{{ }}` nel template | Correggere il riferimento nel template al campo corretto |
| `bad substitution` nello stage Jenkins, con `${params.X}` riportato letteralmente nell'errore | Uso di virgolette singole (`'...'`) in un comando `sh` che richiede interpolazione Groovy | Verificare il tipo di virgolette usato nello step `sh` | Usare virgolette doppie per interpolazione Groovy (`"${params.X}"`), oppure referenziare la variabile d'ambiente shell-style (`$X`) con virgolette singole |
| `ErrImagePull` / `ImagePullBackOff` sul Pod | Il tag specificato in `IMAGE_TAG` non esiste sul registry Docker Hub | `kubectl describe pod <nome> -n formazione-sou`, sezione Events | Specificare un tag realmente pubblicato; non confondere il tag immagine con la versione del chart (`Chart.yaml` → `version`) |
| `helm install` fallisce con "release già esistente" | Si è usato `helm install` invece di `helm upgrade --install` su una release preesistente | `helm list -n formazione-sou` | Usare sempre `helm upgrade --install` nella pipeline per garantire idempotenza |
| La VM Jenkins non riesce a contattare l'API server di Kubernetes | Il tunnel `kubectl proxy` non è attivo, o ascolta solo su `localhost` | `curl http://192.168.56.1:8001/version` dalla VM | Avviare `kubectl proxy` con `--address` impostato sull'interfaccia di rete condivisa con la VM, non lasciare il default |
