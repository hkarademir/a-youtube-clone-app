pipeline {
    agent {
        node {
            label 'Slave'
        }
    }
    tools {
        jdk 'Java17'
        nodejs 'NodeJS16'
    }
    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'
        REGISTRY = 'harbor.y4test.local'
        PROJECT_NAME = 'youtube-cicd'
        APP_NAME = 'youtube-cicd-pipeline'
        RELEASE = '0.0.1'
        DOCKER_PASS = 'jenkins-harbor-user'
        IMAGE_NAME = "${REGISTRY}" + '/' + "${PROJECT_NAME}" + '/' + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    stages {
        stage('Clean Workspace'){
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/hkarademir/a-youtube-clone-app.git'
            }
        }
        stage('Sonarqube Analysis'){
            steps {
                withSonarQubeEnv('sonarqube-server'){
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Youtube-CICD \
                    -Dsonar.projectKey=Youtube-CICD'''
                }
            }
        }
        stage('Quality Gate'){
            steps {
                script {
                    waitForQualityGate abortPipelines: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", "${DOCKER_PASS}") {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry("https://${REGISTRY}", "${DOCKER_PASS}") {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
        stage('Deploy to Kubernetes'){
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubernetes', namespace: '', restrictKubeConfigAccess: false, serverUrl: ''){
                            sh 'kubectl -n youtube-cicd delete --all pods'
                            sh 'kubectl apply -f .'
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            sh 'docker logout'
        }
    }
}