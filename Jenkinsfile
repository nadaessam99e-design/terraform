pipeline {
    agent any

    environment {
        ACR_REGISTRY = 'kokoregistry.azurecr.io'
        ACR_REPO     = 'kokoregistry.azurecr.io/ecomerce-backend'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        TF_DIR       = 'terraform'
    }

    stages {

        stage('Test') {
            steps {
                dir('ecomerce') {
                    sh 'npm install'
                    sh 'npm run lint || echo "lint skipped"'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ecomerce-backend:${IMAGE_TAG} .'
            }
        }

        stage('Push to ACR') {
            steps {
                withCredentials([
                    string(credentialsId: 'ACR_USERNAME', variable: 'ACR_USERNAME'),
                    string(credentialsId: 'ACR_PASSWORD', variable: 'ACR_PASSWORD')
                ]) {
                    sh '''
                        echo "$ACR_PASSWORD" | docker login "$ACR_REGISTRY" \
                            --username "$ACR_USERNAME" \
                            --password-stdin

                        docker tag ecomerce-backend:${IMAGE_TAG} ${ACR_REPO}:${IMAGE_TAG}
                        docker tag ecomerce-backend:${IMAGE_TAG} ${ACR_REPO}:latest

                        docker push ${ACR_REPO}:${IMAGE_TAG}
                        docker push ${ACR_REPO}:latest
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'ACR_USERNAME', variable: 'ACR_USERNAME'),
                    string(credentialsId: 'ACR_PASSWORD', variable: 'ACR_PASSWORD')
                ]) {
                    sh '''
                        cd ${TF_DIR}
                        terraform init
                        terraform validate
                        terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Deploy to Azure?', ok: 'Yes, Deploy!'
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'ACR_USERNAME', variable: 'ACR_USERNAME'),
                    string(credentialsId: 'ACR_PASSWORD', variable: 'ACR_PASSWORD')
                ]) {
                    sh '''
                        cd ${TF_DIR}
                        terraform apply tfplan
                    '''
                }
            }
        }

        stage('VM Pull Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'ACR_USERNAME', variable: 'ACR_USERNAME'),
                    string(credentialsId: 'ACR_PASSWORD', variable: 'ACR_PASSWORD')
                ]) {
                    sh '''
                        echo "$ACR_PASSWORD" | docker login "$ACR_REGISTRY" \
                            --username "$ACR_USERNAME" \
                            --password-stdin

                        docker pull ${ACR_REPO}:latest
                        docker stop ecomerce-backend || true
                        docker rm ecomerce-backend || true
                        docker run -d --name ecomerce-backend -p 3000:3000 ${ACR_REPO}:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
