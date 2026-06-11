pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        AWS_REGION     = 'ap-southeast-1'
        AWS_ACCOUNT_ID = credentials('aws-account-id')
        ECR_REPO       = 'myapp'
        IMAGE_TAG      = "${env.GIT_COMMIT ? env.GIT_COMMIT[0..6] : 'local'}-${env.BUILD_NUMBER}"
        ECR_URI        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        EC2_HOST       = credentials('ec2-host')
        COMPOSE_PATH   = '/home/ec2-user/app/docker-compose.yml'
    }

    stages {

        stage('1. Pull Git') {
            steps {
                checkout scm
                sh 'git log --oneline -3'
            }
        }

        stage('2. Build image') {
            steps {
                sh """
                    docker build \
                        -t ${ECR_REPO}:${IMAGE_TAG} \
                        -t ${ECR_REPO}:latest \
                        .
                """
            }
        }

        stage('3. Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} \
                            | docker login --username AWS --password-stdin \
                              ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
                        docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:latest

                        docker push ${ECR_URI}:${IMAGE_TAG}
                        docker push ${ECR_URI}:latest
                    """
                }
            }
        }

        stage('4. Deploy to EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                                export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                                export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                                export AWS_DEFAULT_REGION=${AWS_REGION}

                                aws ecr get-login-password --region ${AWS_REGION} \
                                    | docker login --username AWS --password-stdin \
                                      ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                                cd /home/ec2-user/app
                                export IMAGE_TAG=${IMAGE_TAG}
                                export ECR_URI=${ECR_URI}
                                docker compose pull
                                docker compose up -d --remove-orphans
                                docker compose ps
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${env.IMAGE_TAG} successfully"
        }
        failure {
            echo "Pipeline FAILED — check logs above"
        }
        always {
             script {
                 def ecrRepo  = env.ECR_REPO  ?: 'unknown'
                 def imageTag = env.IMAGE_TAG  ?: 'unknown'
                 def ecrUri   = env.ECR_URI    ?: 'unknown'
                 sh "docker rmi ${ecrRepo}:${imageTag} || true"
                 sh "docker rmi ${ecrUri}:${imageTag}  || true"
             }
        }
    }
}