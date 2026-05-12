pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'reshma0654/bookmyshow:latest'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'kastro-eks'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
               checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'Git_creds', url: 'https://github.com/Reshma-0654/Book-My-Show.git']]) 
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BookMyShow \
                    -Dsonar.projectKey=BookMyShow
                    '''
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

                echo "Checking package.json..."

                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json

                    echo "Installing dependencies..."
                    npm install
                else
                    echo "package.json not found!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                echo "Building Docker Image..."

                docker build --no-cache -t $DOCKER_IMAGE \
                -f bookmyshow-app/Dockerfile bookmyshow-app
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh '''
                echo "Scanning Docker Image..."

                trivy image $DOCKER_IMAGE > trivyimage.txt
                '''
            }
        }

        stage('Docker Login & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {

                        sh '''
                        echo "Pushing Docker Image..."

                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh '''
                echo "Stopping old container..."

                docker stop bookmyshow || true
                docker rm bookmyshow || true

                echo "Running new container..."

                docker run -d \
                --name bookmyshow \
                --restart=always \
                -p 3000:3000 \
                $DOCKER_IMAGE

                echo "Checking running containers..."

                docker ps -a

                echo "Container Logs..."

                sleep 10
                docker logs bookmyshow
                '''
            }
        }

        stage('Configure AWS CLI') {
            steps {
                sh '''
                echo "Checking AWS CLI..."

                aws --version

                echo "Checking AWS Identity..."

                aws sts get-caller-identity
                '''
            }
        }

        stage('Connect to EKS Cluster') {
            steps {
                sh '''
                echo "Updating kubeconfig..."

                aws eks update-kubeconfig \
                --region $AWS_REGION \
                --name $EKS_CLUSTER_NAME

                echo "Checking Kubernetes Nodes..."

                kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                echo "Deploying Kubernetes Resources..."

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml

                echo "Checking Pods..."

                kubectl get pods

                echo "Checking Services..."

                kubectl get svc
                '''
            }
        }
    }

    post {
        always {

            emailext(
                attachLog: true,
                subject: "${currentBuild.result}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <h2>Jenkins Build Notification</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Build Status:</b> ${currentBuild.result}</p>
                <p><b>Build URL:</b> ${env.BUILD_URL}</p>
                """,
                to: 'yourmail@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
