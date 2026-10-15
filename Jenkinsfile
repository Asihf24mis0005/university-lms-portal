pipeline {
    agent any
    
    environment {
        DOCKER_HUB_USER = 'asihf'
        IMAGE_NAME      = 'university-lms-portal'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest ."
                }
            }
        }
        
        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'k8s-config', variable: 'KUBECONFIG')]) {
                    script {
                        sh "sed -i 's|image: .*|image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml"
                        sh "kubectl apply -f deployment.yaml --kubeconfig=\${KUBECONFIG}"
                        sh "kubectl rollout status deployment/university-portal-deployment --kubeconfig=\${KUBECONFIG}"
                    }
                }
            }
        }
    }
    
    post {
        failure {
            withCredentials([file(credentialsId: 'k8s-config', variable: 'KUBECONFIG')]) {
                script {
                    echo "Deployment failed! Initiating automatic rollback to the last stable release..."
                    sh "kubectl rollout undo deployment/university-portal-deployment --kubeconfig=\${KUBECONFIG}"
                }
            }
        }
    }
}
