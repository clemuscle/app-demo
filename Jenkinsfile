pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: python:3.12-slim
    command: ['sleep', 'infinity']
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:latest
    command: ['sleep', 'infinity']
"""
        }
    }

    environment {
        HARBOR_REGISTRY = "harbor.local"
        HARBOR_PROJECT  = "app-demo"
        IMAGE_NAME      = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/app-demo"
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }

    stages {
        stage('Tests') {
            steps {
                container('python') {
                    sh '''
                        pip install -r requirements.txt
                        pip install pytest-cov
                        pytest tests/ --cov=src --cov-report=xml
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'sonar-scanner'
                    }
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

        stage('Build & Push') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor-credentials',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh '''
                            docker login ${HARBOR_REGISTRY} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy via ArgoCD') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/clemuscle/app-demo-gitops.git
                        cd app-demo-gitops
                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" overlays/dev/patch-image.yaml
                        git config user.email "jenkins@local"
                        git config user.name "Jenkins"
                        git add .
                        git commit -m "ci: update image to ${IMAGE_TAG}"
                        git push
                    '''
                }
            }
        }
    }

    post {
        always {
            junit '**/test-results/*.xml'
        }
        success {
            echo "Pipeline réussi - Image: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline échoué"
        }
    }
}