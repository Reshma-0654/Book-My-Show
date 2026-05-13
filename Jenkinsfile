pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'bindusravya/bms:latest'
        EKS_CLUSTER_NAME = 'kastro-eks'
        AWS_REGION = 'us-east-1'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'github_creds',
                        url: 'https://github.com/Reshma-0654/Book-My-Show.git'
                    ]]
                )
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                rm -rf node_modules package-lock.json
                npm install
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt || true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh '''
                        echo "Logging into DockerHub..."
                        echo $PASS | docker login -u $USER --password-stdin

                        echo "Building Docker image..."
                        docker build -t $DOCKER_IMAGE -f bookmyshow-app/Dockerfile bookmyshow-app

                        echo "Pushing Docker image..."
                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_KEY')
                    ]) {

                        sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY
                        aws configure set aws_secret_access_key $AWS_SECRET_KEY
                        aws configure set region $AWS_REGION

                        aws sts get-caller-identity

                        aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

                        kubectl apply -f deployment.yml
                        kubectl apply -f service.yml

                        kubectl get pods
                        kubectl get svc
                        '''
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                echo "Stopping and removing old container..."
                docker stop bms || true
                docker rm bms || true

                echo "Running new container on port 3000..."
                docker run -d --restart=always --name bms -p 3000:3000 bindusravya/bms:latest

                echo "Checking running containers..."
                docker ps -a

                echo "Fetching logs..."
                sleep 5
                docker logs bms
                '''
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "${currentBuild.result}",
                body: """
                Project: ${env.JOB_NAME}<br/>
                Build: ${env.BUILD_NUMBER}<br/>
                URL: ${env.BUILD_URL}
                """,
                to: 'kastrokiran@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
        }
    }
}
