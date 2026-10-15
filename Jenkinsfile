pipeline {
    agent any
    
    environment {
        DOCKER_HUB_USER = 'asihf'
        IMAGE_NAME      = 'university-lms-portal'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }
    
    stages {
        // Redundant clone stage has been removed. Jenkins automatically manages SCM checkouts.
        
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
                // Using 'string' explicitly maps your Secret Text credential to runtime memory
                withCredentials([string(credentialsId: 'k8s-config', variable: 'KUBECONFIG_CONTENT')]) {
                    script {
                        // Writes the configuration content directly into a workspace config file
                        writeFile file: 'kubeconfig.yaml', text: KUBECONFIG_CONTENT
                        
                        // Updates the deployment tag to match this current build number
                        sh "sed -i 's|image: .*|image: ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml"
                        
                        // Executes the deployment manifests on your local cluster context
                        sh "kubectl apply -f deployment.yaml --kubeconfig=kubeconfig.yaml"
                        sh "kubectl rollout status deployment/university-portal-deployment --kubeconfig=kubeconfig.yaml"
                    }
                }
            }
        }
    }
    
    post {
        failure {
            withCredentials([string(credentialsId: 'k8s-config', variable: 'KUBECONFIG_CONTENT')]) {
                script {
                    echo "Deployment unhealthy! Reverting to the last working application tag..."
                    writeFile file: 'kubeconfig.yaml', text: KUBECONFIG_CONTENT
                    sh "kubectl rollout undo deployment/university-portal-deployment --kubeconfig=kubeconfig.yaml"
                }
            }
        }
    }
}
