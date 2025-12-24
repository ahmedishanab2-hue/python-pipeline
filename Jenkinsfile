pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/MohamedMagdy840/jenkins-repo.git',
                    credentialsId: 'github'
            }
        }

        stage('Run Hello World') {
            steps {
                sh 'ls -l'
                sh 'chmod +x hello.sh'
                sh './hello.sh'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
