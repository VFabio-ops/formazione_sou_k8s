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
    }
    stages {
        stage('Checkout SCM'){
            steps{
                checkout scm 
            }
        }
        stage('Build Image') {
            steps {
                sh "podman build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -f /Esercitazioni/DockerHello/"
                sh "podman tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }
        stage('Push to DockerHub') {
            steps {
                sh "podman login docker.io -u ${REGISTRY_CREDS_USR} --password-stdin <<< ${REGISTRY_CREDS_PSW}"
                sh "podman push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "podman push ${DOCKER_IMAGE}:latest"
            }
        }
    }
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