
pipeline {
    agent any

    parameters {
        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch to build and deploy'
        )

        string(
            name: 'APP_NAME',
            defaultValue: 'new-app-hello-nti',
            description: 'Application and Docker image name'
        )

        string(
            name: 'CONTAINER_PORT',
            defaultValue: '5000',
            description: 'Port used by the application inside the container'
        )

        string(
            name: 'PORT_START',
            defaultValue: '5000',
            description: 'Start of host port range'
        )

        string(
            name: 'PORT_END',
            defaultValue: '5999',
            description: 'End of host port range'
        )

        string(
            name: 'REGISTRY3_HOST',
            defaultValue: 'registry.example.com',
            description: 'Third registry hostname'
        )

        string(
            name: 'REGISTRY3_NAMESPACE',
            defaultValue: 'my-team',
            description: 'Namespace or project in the third registry'
        )
    }

    environment {
        REPO_URL = 'https://github.com/MohamedMagdy840/jenkins-repo.git'

        // Jenkins credential IDs
        DOCKER_CREDENTIALS = 'docker'
        GHCR_CREDENTIALS = 'ghcr'
        REGISTRY3_CREDENTIALS = 'registry3'
    }

    stages {

        stage('Getting Repo Files') {
            steps {
                echo "Building branch: ${params.GIT_BRANCH}"

                git branch: "${params.GIT_BRANCH}",
                    credentialsId: 'jenkins-token',
                    url: "${REPO_URL}"
            }
        }

        stage('Prepare Application') {
            steps {
                script {
                    // Make branch safe for Docker tags and container names
                    env.SAFE_BRANCH = params.GIT_BRANCH
                        .replaceAll('[^a-zA-Z0-9_.-]', '-')
                        .toLowerCase()

                    env.IMAGE_TAG = "${env.SAFE_BRANCH}-${env.BUILD_NUMBER}"

                    env.LOCAL_IMAGE =
                        "${params.APP_NAME}:${env.IMAGE_TAG}"

                    env.CONTAINER_NAME =
                        "${params.APP_NAME}-${env.SAFE_BRANCH}-${env.BUILD_NUMBER}"

                    echo "Application: ${params.APP_NAME}"
                    echo "Branch tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Find Free Host Port') {
            steps {
                script {
                    env.APP_PORT = sh(
                        script: '''
                            python3 - "$PORT_START" "$PORT_END" <<'PY'
import socket
import random
import sys

start = int(sys.argv[1])
end = int(sys.argv[2])

if not (1024 <= start <= end <= 65535):
    raise SystemExit("Invalid port range. Use ports 1024-65535.")

free_ports = []

for port in range(start, end + 1):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        sock.bind(("0.0.0.0", port))
        free_ports.append(port)
    except OSError:
        pass
    finally:
        sock.close()

if not free_ports:
    raise SystemExit("No free ports found in the requested range.")

print(random.choice(free_ports))
PY
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Selected host port: ${env.APP_PORT}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        -t "$LOCAL_IMAGE" \
                        .
                '''
            }
        }

        stage('Tag Image for 3 Registries') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ),
                    usernamePassword(
                        credentialsId: 'ghcr',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_PASS'
                    ),
                    usernamePassword(
                        credentialsId: 'registry3',
                        usernameVariable: 'REGISTRY3_USER',
                        passwordVariable: 'REGISTRY3_PASS'
                    )
                ]) {
                    sh '''
                        set -eu

                        docker tag "$LOCAL_IMAGE" \
                            "$DOCKER_USER/$APP_NAME:$IMAGE_TAG"

                        docker tag "$LOCAL_IMAGE" \
                            "ghcr.io/$GHCR_USER/$APP_NAME:$IMAGE_TAG"

                        docker tag "$LOCAL_IMAGE" \
                            "$REGISTRY3_HOST/$REGISTRY3_NAMESPACE/$APP_NAME:$IMAGE_TAG"

                        echo "Image tagged for all three registries."
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker push \
                            "$DOCKER_USER/$APP_NAME:$IMAGE_TAG"

                        docker logout || true
                    '''
                }
            }
        }

        stage('Push to GitHub Container Registry') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        echo "$GHCR_PASS" | docker login ghcr.io \
                            -u "$GHCR_USER" \
                            --password-stdin

                        docker push \
                            "ghcr.io/$GHCR_USER/$APP_NAME:$IMAGE_TAG"

                        docker logout ghcr.io || true
                    '''
                }
            }
        }

        stage('Push to Third Registry') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'registry3',
                        usernameVariable: 'REGISTRY3_USER',
                        passwordVariable: 'REGISTRY3_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        echo "$REGISTRY3_PASS" | docker login \
                            "$REGISTRY3_HOST" \
                            -u "$REGISTRY3_USER" \
                            --password-stdin

                        docker push \
                            "$REGISTRY3_HOST/$REGISTRY3_NAMESPACE/$APP_NAME:$IMAGE_TAG"

                        docker logout "$REGISTRY3_HOST" || true
                    '''
                }
            }
        }

        stage('Run New Docker Container') {
            steps {
                sh '''
                    set -eu

                    docker rm -f "$CONTAINER_NAME" 2>/dev/null || true

                    docker run -d \
                        --restart unless-stopped \
                        -p "$APP_PORT:$CONTAINER_PORT" \
                        --name "$CONTAINER_NAME" \
                        "$LOCAL_IMAGE"

                    echo "Container status:"
                    docker ps --filter "name=$CONTAINER_NAME"

                    echo "Application URL:"
                    echo "http://localhost:$APP_PORT"
                '''
            }
        }
    }

    post {
        success {
            echo """
            Pipeline completed successfully!

            Branch: ${params.GIT_BRANCH}
            Application: ${params.APP_NAME}
            Image tag: ${env.IMAGE_TAG}
            Host port: ${env.APP_PORT}

            Registries:
            - Docker Hub
            - GitHub Container Registry
            - ${params.REGISTRY3_HOST}
            """
        }

        failure {
            echo 'Pipeline failed. Check the failed stage in the Jenkins console.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
