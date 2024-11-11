pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/MohamedMagdy840/jenkins-repo.git'
  // Replace with your repository URL
            }
        }
        stage('Run Hello World') {
            steps {
                script {
                    // Assuming the 'hello.sh' script is in the root of the repository
                    sh 'chmod +x hello.sh'  // Make the script executable (if not already)
                    sh './hello.sh'          // Run the Hello World script
                }
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
