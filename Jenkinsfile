pipeline {
    agent any

    environment {
        APP_NAME = 'app-hello-nti'
        REPO_URL = 'https://github.com/MohamedMagdy840/jenkins-repo.git'
        APP_PORT = '5000'
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: 'main', credentialsId: 'jenkins-token', url: "${REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${APP_NAME}:${BUILD_NUMBER} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword( usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                            docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}
                            docker push ${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    sh """
                        docker rm -f ${APP_NAME}-${BUILD_NUMBER} 2>/dev/null || true
                        docker run -d -p ${APP_PORT}:${APP_PORT} --name ${APP_NAME}-${BUILD_NUMBER} ${APP_NAME}:${BUILD_NUMBER}
                        docker ps --filter name=${APP_NAME}-${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Image pushed and container running on port ${APP_PORT}"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
