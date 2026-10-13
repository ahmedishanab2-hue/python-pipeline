def pushToRegistry(String registry, String credsId) {
    withCredentials([usernamePassword(
        credentialsId: credsId,
        usernameVariable: 'REG_USER',
        passwordVariable: 'REG_PASS'
    )]) {
        withEnv(["REGISTRY=${registry}"]) {
            sh '''#!/bin/bash
                set +x
                # registries require lowercase repo paths
                USER_LC=$(echo "$REG_USER" | tr '[:upper:]' '[:lower:]')
                TARGET="${REGISTRY}/${USER_LC}/${APP_NAME}:${IMAGE_TAG}"

                echo "$REG_PASS" | docker login "$REGISTRY" -u "$REG_USER" --password-stdin
                docker tag "${APP_NAME}:${IMAGE_TAG}" "$TARGET"
                docker push "$TARGET"
                docker logout "$REGISTRY"
            '''
        }
    }
}

pipeline {
    agent any

    parameters {
        string(name: 'APP_NAME', defaultValue: 'app-hello-nti', description: 'Application name (change for a new app)')
        string(name: 'GIT_BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
        string(name: 'REPO_URL', defaultValue: 'https://github.com/MohamedMagdy840/jenkins-repo.git', description: 'Git repo')
    }

    environment {
        APP_NAME  = "${params.APP_NAME}"
        REPO_URL  = "${params.REPO_URL}"
        APP_PORT  = '5000'            // port inside the container
        PORT_MIN  = '5000'
        PORT_MAX  = '5999'
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: "${params.GIT_BRANCH_NAME}",
                    credentialsId: 'jenkins-token',
                    url: "${REPO_URL}"

                script {
                    // feature/login -> feature-login (valid for docker tags/names)
                    env.BRANCH_SAFE = params.GIT_BRANCH_NAME
                        .replaceAll('[^a-zA-Z0-9_.-]', '-')
                        .replaceAll('^[^a-zA-Z0-9]+', '')   // container names must start with a letter or digit
                        .toLowerCase()
                    env.IMAGE_TAG      = "${env.BRANCH_SAFE}-${env.BUILD_NUMBER}"
                    env.CONTAINER_NAME = "${env.BRANCH_SAFE}-${env.APP_NAME}"   // branch first, then app name
                }
            }
        }

        stage('Find Free Port') {
            steps {
                script {
                    env.HOST_PORT = sh(returnStdout: true, script: '''#!/bin/bash
                        # shuffle the whole range, return the first port that is free
                        for p in $(shuf -i ${PORT_MIN}-${PORT_MAX}); do
                            in_os=$(ss -ltn | awk '{print $4}' | grep -E ":${p}$" || true)
                            in_docker=$(docker ps --format '{{.Ports}}' | grep -E ":${p}->" || true)
                            if [ -z "$in_os" ] && [ -z "$in_docker" ]; then
                                echo $p
                                exit 0
                            fi
                        done
                        echo "No free port found" >&2
                        exit 1
                    ''').trim()
                    echo "Selected free port: ${env.HOST_PORT}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${APP_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Push to Docker Hub') {
            steps { script { pushToRegistry('docker.io', 'docker') } }
        }

        stage('Push to GHCR (GitHub)') {
            steps { script { pushToRegistry('ghcr.io', 'ghcr-creds') } }
        }

        stage('Push to Quay.io') {
            steps { script { pushToRegistry('quay.io', 'quay-creds') } }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    docker rm -f "${CONTAINER_NAME}" 2>/dev/null || true

                    docker run -d \
                        -p "${HOST_PORT}:${APP_PORT}" \
                        --name "${CONTAINER_NAME}" \
                        "${APP_NAME}:${IMAGE_TAG}"

                    docker ps --filter "name=${CONTAINER_NAME}"
                '''
            }
        }
    }

    post {
        success {
            echo "Branch '${params.GIT_BRANCH_NAME}' pushed to 3 registries, container '${env.CONTAINER_NAME}' running at http://<server>:${env.HOST_PORT}"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
