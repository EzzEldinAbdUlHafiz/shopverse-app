pipeline {
    agent {
        label 'docker'
    }

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = 'your-account-id.dkr.ecr.us-east-1.amazonaws.com'
        REPO_NAME = 'shopverse-repo'
        IMAGE_TAG = "prod-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Security Scan') {
            steps {
                script {
                    echo 'Running Trivy vulnerability scan on codebase...'
                    // Use the cached DB configured via Ansible
                    sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL . '
                }
            }
        }

        stage('Build & Push') {
            steps {
                script {
                    echo 'Building Docker images for production...'
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"

                    // Build Backend
                    sh "docker build -t ${ECR_REGISTRY}/${REPO_NAME}:${IMAGE_TAG} ./backend"
                    sh "docker push ${ECR_REGISTRY}/${REPO_NAME}:${IMAGE_TAG}"

                    // Build Frontend
                    sh "docker build -t ${ECR_REGISTRY}/${REPO_NAME}-frontend:${IMAGE_TAG} ./frontend"
                    sh "docker push ${ECR_REGISTRY}/${REPO_NAME}-frontend:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Manifest') {
            steps {
                script {
                    echo 'Updating k8s-config-repo image tag...'
                    sh """
                      git clone https://github.com/YOUR_USER/k8s-config-repo.git
                      cd k8s-config-repo
                      # Update backend tag
                      sed -i 's|backend.image.tag:.*|backend.image.tag: ${IMAGE_TAG}|' helm/shopverse/values-prod.yaml
                      # Update frontend tag
                      sed -i 's|frontend.image.tag:.*|frontend.image.tag: ${IMAGE_TAG}|' helm/shopverse/values-prod.yaml

                      git config user.email "jenkins@shopverse.io"
                      git config user.name "Jenkins CI"
                      git add helm/shopverse/values-prod.yaml
                      git commit -m "ci: update prod image tag to ${IMAGE_TAG} [skip ci]"
                      git push origin main
                    """
                }
            }
        }

        stage('Approval Gate') {
            steps {
                input message: 'Deploy to production?', ok: 'Approve'
            }
        }

        stage('Sync ArgoCD') {
            steps {
                script {
                    echo 'Triggering ArgoCD sync...'
                    // Using argocd CLI configured on Jenkins server
                    sh "argocd app sync shopverse-prod"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo 'Pipeline failed. Check logs for security vulnerabilities or build errors.'
        }
    }
}
