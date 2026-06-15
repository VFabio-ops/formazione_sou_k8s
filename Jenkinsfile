pipeline {
    // 1. AGENT: dove viene eseguita la pipeline
    //    'any' = usa qualsiasi agent disponibile
    //    puoi anche specificare: agent { label 'linux' } o agent { docker 'node:18' }
    agent any

    // 2. ENVIRONMENT: variabili disponibili in tutta la pipeline
    //    withCredentials recupera le credenziali salvate in Jenkins
    environment {
        DOCKER_IMAGE   = "fabioviscusi/hello-world"
        DOCKER_TAG     = "${BUILD_NUMBER}"   // variabile Jenkins automatica
        REGISTRY_CREDS = credentials('dockerhub-credentials')  // ID credenziale Jenkins
        LD_LIBRARY_PATH = "/usr/lib64:${env.LD_LIBRARY_PATH}"
    }

    stages {
        
        // 3. STAGE Checkout: scarica il codice sorgente
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 4. STAGE Build: costruisce l'immagine Docker
        stage('Build Image') {
            steps {
                sh "LD_LIBRARY_PATH=/usr/lib64 podman build -t ${DOCKER_IMAGE}:${DOCKER_TAG} Esercitazioni/DockerHello/"
                sh "LD_LIBRARY_PATH=/usr/lib64 podman tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh "LD_LIBRARY_PATH=/usr/lib64 podman login docker.io -u ${REGISTRY_CREDS_USR} --password-stdin <<< ${REGISTRY_CREDS_PSW}"
                sh "LD_LIBRARY_PATH=/usr/lib64 podman push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "LD_LIBRARY_PATH=/usr/lib64 podman push ${DOCKER_IMAGE}:latest"
            }
        }
    }
    // 6. POST: cosa fare dopo, indipendentemente dall'esito
    post {
        always {
            sh "podman logout docker.io"
            sh "podman rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"  // pulizia immagini locali
           }
        success {
            echo "Build e push completati: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "Pipeline fallita. Controlla i log sopra."
        }
    }
}