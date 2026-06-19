# Hello World - Flask CI/CD Demo

Applicazione Flask minimale ("Hello World") containerizzata con Docker e distribuita tramite una pipeline CI/CD Jenkins basata su Podman.

Il progetto è pensato come esempio didattico per illustrare un flusso completo: scrittura dell'applicazione, containerizzazione, build automatizzata e deploy su un registry Docker.

## Indice

- [Panoramica](#panoramica)
- [Struttura del repository](#struttura-del-repository)
- [Requisiti](#requisiti)
- [Avvio in locale](#avvio-in-locale)
- [Build e avvio con Docker](#build-e-avvio-con-docker)
- [Pipeline CI/CD](#pipeline-cicd)
- [Configurazione delle credenziali Jenkins](#configurazione-delle-credenziali-jenkins)
- [Licenza](#licenza)

## Panoramica

L'applicazione esposta da `app.py` è un server Flask che risponde con il testo "Hello World!" sulla rotta principale (`/`). È pensata come base di partenza per testare il ciclo completo di containerizzazione e automazione del deploy.

## Struttura del repository

```
.
├── app.py          # Applicazione Flask
├── Dockerfile      # Definizione dell'immagine container
├── Jenkinsfile     # Pipeline CI/CD
├── LICENSE         # Licenza del progetto
└── README.md
```

## Requisiti

- Python 3.10 o superiore
- Flask
- Docker oppure Podman
- Jenkins con un agent configurato con il label `podman-agent-01` (per l'esecuzione della pipeline)

## Avvio in locale

Clonare il repository e installare le dipendenze:

```bash
git clone <url-del-repository>
cd <nome-cartella>
pip install flask
python3 app.py
```

L'applicazione sarà disponibile su [http://localhost:8000](http://localhost:8000).

## Build e avvio con Docker

Per costruire l'immagine:

```bash
docker build -t hello-world .
```

Per avviare il container:

```bash
docker run -d -p 8000:8000 hello-world
```

L'applicazione sarà accessibile su [http://localhost:8000](http://localhost:8000).

## Pipeline CI/CD

Il file `Jenkinsfile` definisce una pipeline che esegue automaticamente i seguenti passaggi:

1. **Checking Date** – verifica che la build non venga eseguita durante il fine settimana.
2. **Build Image** – costruisce l'immagine con Podman e la marca con il numero di build e con il tag `latest`.
3. **Push to DockerHub** – effettua il login su DockerHub e pubblica entrambe le immagini.
4. **Run Podman in Podman** – avvia un container basato sull'immagine appena pubblicata, esponendo la porta 8000.

Al termine della pipeline, indipendentemente dall'esito, viene eseguito il logout da DockerHub e la rimozione dell'immagine locale legata al numero di build.

La pipeline richiede un agent Jenkins con il label `podman-agent-01` e l'accesso al comando `podman`.

## Configurazione delle credenziali Jenkins

La pipeline utilizza una credenziale Jenkins di tipo *Username with password* per autenticarsi su DockerHub. È necessario configurarla con l'identificativo:

```
dockerhub-credentials
```

Questa credenziale deve contenere lo username e una password o access token validi per il proprio account DockerHub.

## Licenza

Questo progetto è distribuito sotto licenza MIT. Per i dettagli completi consultare il file [LICENSE](LICENSE).