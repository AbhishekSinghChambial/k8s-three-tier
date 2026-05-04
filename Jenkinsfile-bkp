pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDS = credentials('dockerhub-creds')
        DOCKER_USERNAME = 'abhisheksingh143'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/AbhishekSinghChambial/k8s-three-tier.git'
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh "docker build -t ${DOCKER_USERNAME}/frontend:${IMAGE_TAG} ."
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                dir('backend') {
                    sh "docker build -t ${DOCKER_USERNAME}/backend:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin"
                sh "docker push ${DOCKER_USERNAME}/frontend:${IMAGE_TAG}"
                sh "docker push ${DOCKER_USERNAME}/backend:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl set image deployment/frontend frontend=${DOCKER_USERNAME}/frontend:${IMAGE_TAG} -n three-tier"
                sh "kubectl set image deployment/backend backend=${DOCKER_USERNAME}/backend:${IMAGE_TAG} -n three-tier"
                sh "kubectl rollout status deployment/frontend -n three-tier"
                sh "kubectl rollout status deployment/backend -n three-tier"
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline successful! App deployed.'
        }
        failure {
            echo '❌ Pipeline failed! Check logs.'
        }
    }
}

