pipeline {
    agent any

    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'sonar-scanner'
    }

    environment {
        DOCKER_HUB_CREDS = credentials('dockerhub-creds')
        DOCKER_USERNAME = 'abhisheksingh143'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        WORKSPACE_PATH = "/var/jenkins_home/workspace/three-tier-pipeline"
    }

    stages {

        // ✅ PRE-FLIGHT CHECKS - Pehle sab check karo
        stage('Pre-Flight Checks') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "🔍 PRE-FLIGHT CHECKS START"
                    echo "=========================================="

                    echo ""
                    echo "📁 Workspace path:"
                    pwd
                    echo ""

                    echo "👤 Running as user:"
                    whoami
                    id
                    echo ""

                    echo "🐳 Docker availability:"
                    docker version --format "Client: {{.Client.Version}} | Server: {{.Server.Version}}"
                    echo ""

                    echo "📝 Workspace write permission test:"
                    touch /var/jenkins_home/workspace/three-tier-pipeline/permission_test.txt && \
                        echo "✅ Write OK" && \
                        rm /var/jenkins_home/workspace/three-tier-pipeline/permission_test.txt || \
                        echo "❌ WRITE FAILED - fix permissions!"
                    echo ""

                    echo "🔌 Trivy server reachability:"
                    curl -sf http://host.docker.internal:4954/healthz && \
                        echo "✅ Trivy server UP" || \
                        echo "⚠️  Trivy server not reachable at :4954 - will use standalone mode"
                    echo ""

                    echo "🔌 SonarQube reachability:"
                    curl -sf http://host.docker.internal:9000/api/system/status && \
                        echo "" && echo "✅ SonarQube UP" || \
                        echo "⚠️  SonarQube not reachable"
                    echo ""

                    echo "☸️  kubectl check:"
                    kubectl version --client --short 2>/dev/null || kubectl version --client
                    kubectl get nodes 2>&1 | head -5
                    echo ""

                    echo "=========================================="
                    echo "✅ PRE-FLIGHT CHECKS DONE"
                    echo "=========================================="
                '''
            }
        }

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
                    echo "🔍 Running Trivy scan on Frontend image..."
                    echo "📂 Using workspace: ${WORKSPACE_PATH}"

                    # Ensure output dir exists and is writable
                    mkdir -p ${WORKSPACE_PATH}
                    chmod 777 ${WORKSPACE_PATH}

                    docker run --rm \\
                        -v ${WORKSPACE_PATH}:/reports \\
                        aquasec/trivy:latest image \\
                        --server http://host.docker.internal:4954 \\
                        --severity HIGH,CRITICAL \\
                        --exit-code 0 \\
                        --format json \\
                        --output /reports/trivy-frontend-report.json \\
                        ${DOCKER_USERNAME}/frontend:${IMAGE_TAG}

                    echo "✅ Frontend scan done. Checking file..."
                    ls -lh ${WORKSPACE_PATH}/trivy-frontend-report.json || echo "❌ Frontend report NOT found!"
                """

                sh """
                    echo "🔍 Running Trivy scan on Backend image..."

                    docker run --rm \\
                        -v ${WORKSPACE_PATH}:/reports \\
                        aquasec/trivy:latest image \\
                        --server http://host.docker.internal:4954 \\
                        --severity HIGH,CRITICAL \\
                        --exit-code 0 \\
                        --format json \\
                        --output /reports/trivy-backend-report.json \\
                        ${DOCKER_USERNAME}/backend:${IMAGE_TAG}

                    echo "✅ Backend scan done. Checking file..."
                    ls -lh ${WORKSPACE_PATH}/trivy-backend-report.json || echo "❌ Backend report NOT found!"
                """

                sh """
                    echo "📋 Final report listing:"
                    ls -lh ${WORKSPACE_PATH}/trivy-*.json || echo "No trivy reports found in workspace!"

                    echo ""
                    echo "🔢 Frontend vulnerability summary:"
                    cat ${WORKSPACE_PATH}/trivy-frontend-report.json | \
                        python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    results = data.get('Results', [])
    total_high = sum(
        1 for r in results
        for v in r.get('Vulnerabilities', []) or []
        if v.get('Severity') == 'HIGH'
    )
    total_crit = sum(
        1 for r in results
        for v in r.get('Vulnerabilities', []) or []
        if v.get('Severity') == 'CRITICAL'
    )
    print(f'  CRITICAL: {total_crit}  |  HIGH: {total_high}')
except Exception as e:
    print(f'  Could not parse: {e}')
" || true

                    echo ""
                    echo "🔢 Backend vulnerability summary:"
                    cat ${WORKSPACE_PATH}/trivy-backend-report.json | \
                        python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    results = data.get('Results', [])
    total_high = sum(
        1 for r in results
        for v in r.get('Vulnerabilities', []) or []
        if v.get('Severity') == 'HIGH'
    )
    total_crit = sum(
        1 for r in results
        for v in r.get('Vulnerabilities', []) or []
        if v.get('Severity') == 'CRITICAL'
    )
    print(f'  CRITICAL: {total_crit}  |  HIGH: {total_high}')
except Exception as e:
    print(f'  Could not parse: {e}')
" || true
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-*.json',
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

