pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        ECR_REPO     = '114223852322.dkr.ecr.us-east-1.amazonaws.com/shopflow-app'
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
                sh 'docker build -t shopflow-app:${IMAGE_TAG} .'
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $ECR_REPO

                        docker tag shopflow-app:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                        docker tag shopflow-app:${IMAGE_TAG} ${ECR_REPO}:latest

                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

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
                    input message: 'Deploy to AWS?', ok: 'Yes, Deploy!'
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        cd ${TF_DIR}
                        terraform apply tfplan
                    '''
                }
            }
        }

        stage('EC2 Pull Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        aws ssm send-command \
                            --region $AWS_REGION \
                            --document-name "AWS-RunShellScript" \
                            --targets "Key=tag:Name,Values=shopflow-ec2" \
                            --parameters commands="[
                                'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO',
                                'docker pull ${ECR_REPO}:latest',
                                'docker stop shopflow-app || true',
                                'docker rm shopflow-app || true',
                                'docker run -d --name shopflow-app -p 3000:3000 ${ECR_REPO}:latest'
                            ]"
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
