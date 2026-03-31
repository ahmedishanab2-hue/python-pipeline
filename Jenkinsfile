pipeline {
    agent any
    
    environment {
        APP_NAME = 'web-app'
        REPO_URL ="https://github.com/MohamedMagdy840/jenkins-repo.git"
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: "${GIT_BRANCH}", credentialsId: 'github', url: "${REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image
                    sh 'docker build -t ${APP_NAME}:${BUILD_NUMBER} .'
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    // Run Docker image with build number in container name
                    sh """
                        docker run --name "${APP_NAME}"-"${GIT_BRANCH}"-${BUILD_NUMBER} -d ${APP_NAME}:${BUILD_NUMBER}
                        docker ps
                    """
                }
            }
        }
    }
}
