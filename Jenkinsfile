pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'your-dockerhub-username/cicd-automation', description: 'DockerHub image name')
        string(name: 'APP_SERVER_HOST', defaultValue: 'app-server.example.com', description: 'App server SSH host (user@host)')
        string(name: 'SNS_TOPIC_ARN', defaultValue: 'arn:aws:sns:REGION:ACCOUNT_ID:JenkinsPipelineNotifications', description: 'SNS topic ARN for pipeline notifications')
    }

    environment {
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Clone') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build --pull -t ${params.IMAGE_NAME}:${IMAGE_TAG} ."
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
                    sh "docker push ${params.IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to EC2 App Server') {
            steps {
                sshagent(['app-server-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${params.APP_SERVER_HOST} '
                            docker pull ${params.IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop app || true &&
                            docker rm app || true &&
                            docker run -d --restart unless-stopped --name app -p 80:80 ${params.IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        stage('Notify Success') {
            steps {
                sh """
                    aws sns publish \
                        --topic-arn "${params.SNS_TOPIC_ARN}" \
                        --message "Deployment successful for Build #${BUILD_NUMBER}"
                """
            }
        }
    }

    post {
        failure {
            sh """
                aws sns publish \
                    --topic-arn "${params.SNS_TOPIC_ARN}" \
                    --message "Deployment failed for Build #${BUILD_NUMBER}"
            """
        }
    }
}
