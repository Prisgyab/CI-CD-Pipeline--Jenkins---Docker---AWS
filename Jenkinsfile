pipeline {
    agent any

    environment {
        IMAGE_NAME = "edwinaabah/cicd-automation"
        IMAGE_TAG  = "build-${BUILD_NUMBER}"

        // SNS Topic ARN
        SNS_TOPIC_ARN = "arn:aws:sns:eu-central-1:506993645576:JenkinsPipelineNotifications"
    }

    stages {

        stage('Clone') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build --pull -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to EC2 App Server') {
            steps {
                sshagent(['app-server-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@63.178.45.75 '
                            docker pull ${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop app || true &&
                            docker rm app || true &&
                            docker run -d --restart unless-stopped --name app -p 80:80 ${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        stage('Notify Success') {
            steps {
                sh """
                    aws sns publish \
                        --topic-arn "${SNS_TOPIC_ARN}" \
                        --message "Deployment successful for Build #${BUILD_NUMBER}"
                """
            }
        }
    }

    post {
        failure {
            sh """
                aws sns publish \
                    --topic-arn "${SNS_TOPIC_ARN}" \
                    --message "Deployment failed for Build #${BUILD_NUMBER}"
            """
        }
    }
}
