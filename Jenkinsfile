pipeline {
    agent any

    environment {
        AWS_REGION         = 'ap-south-1'
        AWS_ACCOUNT_ID     = '754660694286'
        ECR_REPO_NAME      = 'seclock'
        IMAGE_TAG          = "${BUILD_NUMBER}"
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
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --upgrade pip
                    pip install bandit
                    bandit -r . -x ./sample_certificates --severity-level medium
                '''
            }
        }

        stage('SCA - Pip Audit') {
            steps {
                sh '''
                    . .venv/bin/activate
                    pip install pip-audit
                    pip_audit -r requirements.txt
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
                    # Spin up temporary container for dynamic testing
                    docker run -d --name seclock-dast-target -p 8080:8080 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                    
                    # Allow app initialization
                    sleep 5

                    # Run baseline scan against documentation/API endpoint
                    docker run --rm --network="host" -v \$(pwd):/zap/wrk/:rw zaproxy/zap-stable zap-baseline.py \
                        -t http://localhost:8080/docs \
                        -r zap_report.html \
                        -I || true

                    # Clean up testing container
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
                withCredentials([usernamePassword(
                    credentialsId: "${AWS_CREDENTIAL_ID}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                        export AWS_ACCESS_KEY_ID=\$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=\$AWS_SECRET_ACCESS_KEY
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Manifest Tag') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                    passwordVariable: 'GIT_PASSWORD',
                    usernameVariable: 'GIT_USERNAME'
                )]) {
                    sh """
                        sed -i "s|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}|g" seclock-manifest/deployment.yaml
                        git config user.email "jenkins@devsecops.local"
                        git config user.name "Jenkins DevSecOps"
                        git commit -am "chore(deploy): bump image tag to ${IMAGE_TAG} [ci skip]" || echo "No changes to commit"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Beyo04/seclock.git HEAD:main
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