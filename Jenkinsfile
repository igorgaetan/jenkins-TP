pipeline {
    agent any

    environment {
        AWS_REGION      = 'eu-north-1'                         // Adaptez à votre région
        ECR_REGISTRY    = '172030247215.dkr.ecr.eu-north-1.amazonaws.com'  // Votre ID de compte AWS
        ECR_REPO        = 'mon-app-nginx'
        IMAGE_TAG       = "${env.GIT_COMMIT[0..6]}"
        APP_EC2_HOST    = credentials('app-ec2-host')         // IP de l'EC2 applicatif
        APP_EC2_USER    = 'ubuntu'
        CONTAINER_NAME  = 'app-nginx'
        APP_PORT        = '80'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        stage('Build Image') {
            steps {
                script {
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:latest"
                }
            }
        }

        stage('Push vers ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:latest"
                }
            }
        }

        stage('Déployer sur App EC2') {
            steps {
                sshagent(credentials: ['app-ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} '
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker pull ${ECR_REGISTRY}/${ECR_REPO}:latest
                            docker stop ${CONTAINER_NAME} || true
                            docker rm   ${CONTAINER_NAME} || true
                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                -p ${APP_PORT}:80 \
                                --restart unless-stopped \
                                ${ECR_REGISTRY}/${ECR_REPO}:latest
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Déploiement réussi — image : ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ La pipeline a échoué. Consultez les logs ci-dessus."
        }
        always {
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO}:latest       || true"
        }
    }
}