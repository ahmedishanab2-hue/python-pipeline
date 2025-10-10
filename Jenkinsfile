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
                    sh 'chmod +x hello.sh'  
                    sh './hello.sh'          
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
