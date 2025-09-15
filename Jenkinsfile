pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "vishalk15v/book-my-show"
        SONAR_TOKEN  = credentials('SonarQube-Tokenn')
        DOCKER_REGISTRY_CREDENTIALS = 'Vishal-Dockerhub-Credentials'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/VshalRawat/Book-My-Show.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('bookmyshow-app') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            sonar-scanner \
                            -Dsonar.projectKey=BookMyShow \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://13.49.160.95:9000 \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.projectVersion=${env.BUILD_NUMBER} \
                            -Dsonar.sourceEncoding=UTF-8
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

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh 'npm install --legacy-peer-deps'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                dir('bookmyshow-app') {
                    script {
                        withCredentials([usernamePassword(credentialsId: env.DOCKER_REGISTRY_CREDENTIALS, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                        }
                        
                        def customImage = docker.build("${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}", "--build-arg NODE_OPTIONS=--openssl-legacy-provider .")
                        
                        customImage.push()
                        
                        sh "docker tag ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} ${env.DOCKER_IMAGE}:latest"
                        sh "docker push ${env.DOCKER_IMAGE}:latest"
                        
                        sh "docker rmi ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} ${env.DOCKER_IMAGE}:latest || true"
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    dir('bookmyshow-app') {
                      
                        sh """
                            aws eks update-kubeconfig --region ${env.AWS_REGION} --name ${env.EKS_CLUSTER}
                        """

        
                        sh """
                            kubectl apply -f k8s/deployment.yaml -n ${env.K8S_NAMESPACE}
                            kubectl apply -f k8s/service.yaml -n ${env.K8S_NAMESPACE}
                        """

                        sh """
                            kubectl set image deployment/bookmyshow-deployment bookmyshow-container=${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} -n ${env.K8S_NAMESPACE}
                        """


                        sh "kubectl rollout status deployment/bookmyshow-deployment -n ${env.K8S_NAMESPACE}"
                    }
                }
            }
        }

        stage('Deploy to Docker Container') {
            steps {
                script {
                    sh 'docker stop bms_app || true'
                    sh 'docker rm bms_app || true'
                    sh "docker pull ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh """
                        docker run -d \
                        --name bms_app \
                        -p 3000:3000 \
                        -e NODE_OPTIONS=--openssl-legacy-provider \
                        -e PORT=3000 \
                        ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}
                    """
                    sh 'sleep 10'
                    sh 'curl -f http://localhost:3000 || exit 1'
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    sh '''
                        echo "Testing application accessibility..."
                        response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000)
                        if [ "$response" -eq 200 ]; then
                            echo "Application is running successfully!"
                        else
                            echo "Application test failed with HTTP code: $response"
                            exit 1
                        fi
                    '''
                }
            }
        }

    

    post {
        always {
            echo "Build ${currentBuild.fullDisplayName} completed with status: ${currentBuild.currentResult}"
            cleanWs()
            sh 'docker logout || true'
        }
        success {
            echo "Build and Deploy Successful!"
        }
        failure {
            echo "Build Failed!"
        }
        unstable {
            echo "Build is unstable!"
        }
    }
}
