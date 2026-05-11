pipeline {
    agent any

    stages {

        stage('Git Checkout') {
            steps {
               checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'Git_creds', url: 'https://github.com/Reshma-0654/Book-My-Show.git']])
            }
        }

    }
}
