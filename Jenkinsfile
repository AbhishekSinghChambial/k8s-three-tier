pipeline {
    agent any

    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'sonar-scanner'
    }

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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/bin/sonar-scanner \
                        -Dsonar.projectKey=three-tier-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://host.docker.internal:9000
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
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

        stage('Trivy Scan') {
            steps {
                sh """
                    docker run --rm \
                    -v \$(pwd):/reports \
                    -v \$(pwd)/html.tpl:/tmp/html.tpl \
                    aquasec/trivy:latest image \
                    --server http://host.docker.internal:4954 \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --format template \
                    --template "@/tmp/html.tpl" \
                    --output /reports/trivy-frontend-report.html \
                    ${DOCKER_USERNAME}/frontend:${IMAGE_TAG}
                """
                sh """
                    docker run --rm \
                    -v \$(pwd):/reports \
                    -v \$(pwd)/html.tpl:/tmp/html.tpl \
                    aquasec/trivy:latest image \
                    --server http://host.docker.internal:4954 \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --format template \
                    --template "@/tmp/html.tpl" \
                    --output /reports/trivy-backend-report.html \
                    ${DOCKER_USERNAME}/backend:${IMAGE_TAG}
                """
                sh "ls -la trivy-*.html || true"
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-*.html',
                        allowEmptyArchive: true,
                        fingerprint: true
                }
            }
        }

        stage('Update YAML and Deploy') {
            steps {
                sh "sed -i 's|${DOCKER_USERNAME}/frontend:.*|${DOCKER_USERNAME}/frontend:${IMAGE_TAG}|g' k8s/frontend-deployment.yaml"
                sh "sed -i 's|${DOCKER_USERNAME}/backend:.*|${DOCKER_USERNAME}/backend:${IMAGE_TAG}|g' k8s/backend-deployment.yaml"
                sh "kubectl apply -f k8s/frontend-deployment.yaml -n three-tier"
                sh "kubectl apply -f k8s/backend-deployment.yaml -n three-tier"
                sh "kubectl rollout status deployment/frontend -n three-tier"
                sh "kubectl rollout status deployment/backend -n three-tier"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline successful! Deployed: ${IMAGE_TAG}"
        }
        failure {
            echo '❌ Pipeline failed! Check logs.'
        }
    }
}