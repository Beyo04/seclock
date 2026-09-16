pipeline {
    agent any

    environment {
        AWS_REGION         = 'ap-south-1'
        AWS_ACCOUNT_ID     = '754660694286'
        ECR_REPO_NAME      = 'seclock'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        GITOPS_REPO_URL    = 'github.com/Beyo04/seclock-gitops.git'
        GIT_CREDENTIALS_ID = 'github-credentials'
        AWS_CREDENTIAL_ID  = 'aws-ecr-credentials'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SAST - Bandit') {
            steps {
                sh '''
                    python3 -m pip install bandit
                    python3 -m bandit -r . -x ./sample_certificates --severity-level medium
                '''
            }
        }

        stage('SCA - Pip Audit') {
            steps {
                sh '''
                    python3 -m pip install pip-audit
                    python3 -m pip_audit -r requirements.txt
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('DAST - OWASP ZAP Scan') {
            steps {
                sh """
                    docker run -d --name seclock-dast-target -p 8080:8080 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                    sleep 5
                    docker run --rm --network="host" -v \$(pwd):/zap/wrk/:rw zaproxy/zap-stable zap-baseline.py \
                        -t http://localhost:8080/docs \
                        -r zap_report.html \
                        -I || true
                    docker stop seclock-dast-target
                    docker rm seclock-dast-target
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image --exit-code 0 --severity HIGH,CRITICAL ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Image to AWS ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIAL_ID}"
                ]]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        rm -rf gitops-repo
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${GITOPS_REPO_URL} gitops-repo
                        cd gitops-repo/k8s
                        sed -i "s|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}|g" deployment.yaml
                        git config user.email "jenkins@devsecops.local"
                        git config user.name "Jenkins DevSecOps"
                        git commit -am "chore(deploy): bump seclock image to tag ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
