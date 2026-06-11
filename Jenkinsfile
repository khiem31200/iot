pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        AWS_REGION      = "${env.AWS_DEFAULT_REGION ?: 'ap-southeast-1'}"
        AWS_ACCOUNT_ID  = credentials('aws-account-id')
        ECR_REPO        = 'myapp'
        IMAGE_TAG       = "${env.GIT_COMMIT ? env.GIT_COMMIT[0..6] : 'local'}-${env.BUILD_NUMBER}"
        ECR_URI         = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        EC2_HOST        = credentials('ec2-host')
        COMPOSE_PATH    = '/home/ec2-user/app/docker-compose.yml'
    }

    stages {

        // ─────────────────────────────────────────
        stage('1. Pull source') {
        // ─────────────────────────────────────────
            steps {
                echo "Checking out source code..."
                checkout scm
                sh 'git log --oneline -5'
            }
        }

        // ─────────────────────────────────────────
        stage('2. Build image') {
        // ─────────────────────────────────────────
            steps {
                echo "Building Docker image: ${ECR_REPO}:${IMAGE_TAG}"
                sh """
                    docker build \
                        --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                        --build-arg GIT_COMMIT=${IMAGE_TAG} \
                        -t ${ECR_REPO}:${IMAGE_TAG} \
                        -t ${ECR_REPO}:latest \
                        .
                """
                sh "docker images ${ECR_REPO}"
            }
        }

        // ─────────────────────────────────────────
        stage('3. Push to AWS ECR') {
        // ─────────────────────────────────────────
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        echo "Logging in to ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                            | docker login \
                                --username AWS \
                                --password-stdin \
                                ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        echo "Tagging image for ECR..."
                        docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}

                        echo "Pushing to ECR..."
                        docker push ${ECR_URI}:${IMAGE_TAG}
                    """
                }
            }
        }

        // ─────────────────────────────────────────
        stage('4. Create image tag') {
        // ─────────────────────────────────────────
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        echo "Tagging as :latest..."
                        docker tag ${ECR_URI}:${IMAGE_TAG} ${ECR_URI}:latest
                        docker push ${ECR_URI}:latest
                    """
                }
                // Ghi image tag ra file để các stage sau hoặc downstream jobs dùng
                sh "echo ${IMAGE_TAG} > image_tag.txt"
                archiveArtifacts artifacts: 'image_tag.txt'
                echo "Image ready: ${ECR_URI}:${IMAGE_TAG}"
            }
        }

        // ─────────────────────────────────────────
        stage('5. Deploy to EC2') {
        // ─────────────────────────────────────────
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_HOST} bash << 'ENDSSH'
                                set -e

                                echo "=== ECR Login ==="
                                export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                                export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                                export AWS_DEFAULT_REGION=${AWS_REGION}

                                aws ecr get-login-password --region ${AWS_REGION} \
                                    | docker login \
                                        --username AWS \
                                        --password-stdin \
                                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                                echo "=== Update IMAGE_TAG in env ==="
                                cd /home/ec2-user/app
                                sed -i 's|IMAGE_TAG=.*|IMAGE_TAG=${IMAGE_TAG}|g' .env
                                sed -i 's|ECR_URI=.*|ECR_URI=${ECR_URI}|g' .env

                                echo "=== Pull latest image ==="
                                docker compose pull

                                echo "=== Restart containers ==="
                                docker compose up -d --remove-orphans

                                echo "=== Running containers ==="
                                docker compose ps
ENDSSH
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Image: ${ECR_URI}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline FAILED at stage — check logs above"
        }
        always {
            // Xóa image local để tiết kiệm disk
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_URI}:${IMAGE_TAG} || true"
        }
    }
}
