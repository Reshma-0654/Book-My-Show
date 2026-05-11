pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'Git_creds',
                        url: 'https://github.com/Reshma-0654/Book-My-Show.git'
                    ]]
                )
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {

                    sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=bookmyshow \
                    -Dsonar.host.url=http://172.31.40.13:9000 \
                    -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

    }
}
