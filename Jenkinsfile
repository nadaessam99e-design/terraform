pipeline {
    agent any

    environment {
        ACR_REGISTRY  = 'kokoregistry.azurecr.io'
        ACR_REPO      = 'kokoregistry.azurecr.io/shopflow-app'
        IMAGE_TAG     = "${BUILD_NUMBER}"
        TF_DIR        = 'terraform'
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
                sh 'docker build -t shopflow-app:${IMAGE_TAG} .'
            }
        }

        stage('Push to ACR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET',   variable: 'AZURE_CLIENT_SECRET'),
                    string(credentialsId: 'AZURE_TENANT_ID',       variable: 'AZURE_TENANT_ID'),
                    string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        az login --service-principal \
                            --username $AZURE_CLIENT_ID \
                            --password $AZURE_CLIENT_SECRET \
                            --tenant $AZURE_TENANT_ID

                        az account set --subscription $AZURE_SUBSCRIPTION_ID

                        az acr login --name kokoregistry

                        docker tag shopflow-app:${IMAGE_TAG} ${ACR_REPO}:${IMAGE_TAG}
                        docker tag shopflow-app:${IMAGE_TAG} ${ACR_REPO}:latest

                        docker push ${ACR_REPO}:${IMAGE_TAG}
                        docker push ${ACR_REPO}:latest
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET',   variable: 'AZURE_CLIENT_SECRET'),
                    string(credentialsId: 'AZURE_TENANT_ID',       variable: 'AZURE_TENANT_ID'),
                    string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        export ARM_CLIENT_ID=$AZURE_CLIENT_ID
                        export ARM_CLIENT_SECRET=$AZURE_CLIENT_SECRET
                        export ARM_TENANT_ID=$AZURE_TENANT_ID
                        export ARM_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

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
                    string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET',   variable: 'AZURE_CLIENT_SECRET'),
                    string(credentialsId: 'AZURE_TENANT_ID',       variable: 'AZURE_TENANT_ID'),
                    string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        export ARM_CLIENT_ID=$AZURE_CLIENT_ID
                        export ARM_CLIENT_SECRET=$AZURE_CLIENT_SECRET
                        export ARM_TENANT_ID=$AZURE_TENANT_ID
                        export ARM_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

                        cd ${TF_DIR}
                        terraform apply tfplan
                    '''
                }
            }
        }

        stage('VM Pull Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET',   variable: 'AZURE_CLIENT_SECRET'),
                    string(credentialsId: 'AZURE_TENANT_ID',       variable: 'AZURE_TENANT_ID'),
                    string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        az login --service-principal \
                            --username $AZURE_CLIENT_ID \
                            --password $AZURE_CLIENT_SECRET \
                            --tenant $AZURE_TENANT_ID

                        az account set --subscription $AZURE_SUBSCRIPTION_ID

                        az vm run-command invoke \
                            --resource-group shopflow-rg \
                            --name shopflow-vm \
                            --command-id RunShellScript \
                            --scripts "
                                az acr login --name kokoregistry
                                docker pull ${ACR_REPO}:latest
                                docker stop shopflow-app || true
                                docker rm shopflow-app || true
                                docker run -d --name shopflow-app -p 3000:3000 ${ACR_REPO}:latest
                            "
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
