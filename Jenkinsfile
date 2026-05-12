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

        EKS_CLUSTER = 'kastro-eks'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {

            steps {

                checkout scmGit(

                    branches: [[name: '*/main']],

                    userRemoteConfigs: [[
                        credentialsId: 'Git_creds',
                        url: 'https://github.com/Reshma-0654/Book-My-Show.git'
                    ]]
                )
            }
        }

        stage('Install Dependencies') {

            steps {

                dir('bookmyshow-app') {

                    sh '''
                    echo "Removing old dependencies..."

                    rm -rf node_modules
                    rm -rf build
                    rm -rf dist

                    echo "Cleaning npm cache..."

                    npm cache clean --force

                    echo "Installing dependencies..."

                    npm ci --legacy-peer-deps
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {

            steps {

                dir('bookmyshow-app') {

                    withSonarQubeEnv('sonar-server') {

                        sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BookMyShow \
                        -Dsonar.projectKey=BookMyShow \
                        -Dsonar.sources=src \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.exclusions=node_modules/**,build/**,dist/**,coverage/**,.git/**
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {

            steps {

                timeout(time: 5, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy File System Scan') {

            steps {

                sh '''
                echo "Running Trivy File Scan..."

                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {

            steps {

                sh '''
                echo "Building Docker Image..."

                docker build \
                -t $DOCKER_IMAGE \
                -f bookmyshow-app/Dockerfile \
                bookmyshow-app
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

                withDockerRegistry(
                    credentialsId: 'docker'
                ) {

                    sh '''
                    echo "Pushing Docker Image..."

                    docker push $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Deploy Docker Container') {

            steps {

                sh '''
                echo "Stopping old container..."

                docker stop bookmyshow || true

                docker rm bookmyshow || true

                echo "Starting new container..."

                docker run -d \
                --name bookmyshow \
                --restart always \
                -p 3000:3000 \
                $DOCKER_IMAGE

                sleep 15

                echo "Running Containers:"
                docker ps -a

                echo "Container Logs:"
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
                --name $EKS_CLUSTER

                echo "Cluster Nodes..."

                kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {

            steps {

                sh '''
                echo "Deploying to Kubernetes..."

                kubectl apply -f deployment.yml

                kubectl apply -f service.yml

                echo "Pods..."

                kubectl get pods

                echo "Services..."

                kubectl get svc
                '''
            }
        }
    }

    post {

        always {

            sh 'docker system prune -f || true'

            emailext(

                attachLog: true,

                subject: "${currentBuild.currentResult}: Job ${env.JOB_NAME}",

                body: """

                <h2>Jenkins Build Notification</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>

                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>

                <p><b>Status:</b> ${currentBuild.currentResult}</p>

                <p><b>Build URL:</b> ${env.BUILD_URL}</p>

                """,

                to: 'yourmail@gmail.com',

                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
