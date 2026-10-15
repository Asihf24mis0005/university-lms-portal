pipeline {
    agent any
    
    environment {
        DOCKER_HUB_USER = 'asihf'
        IMAGE_NAME      = 'university-lms-portal'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    // Changed from sh to bat for native Windows script execution
                    bat "docker build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
                    bat "docker build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:latest ."
                }
            }
        }
        
        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    // Windows batch script to pipe credentials securely into Docker CLI login
                    bat "echo %DOCKER_PASSWORD%| docker login -u %DOCKER_USERNAME% --password-stdin"
                    bat "docker push %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
                    bat "docker push %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'k8s-config', variable: 'KUBECONFIG_CONTENT')]) {
                    script {
                        // Writes the configuration content directly into a workspace config file
                        writeFile file: 'kubeconfig.yaml', text: KUBECONFIG_CONTENT
                        
                        // Updates image tags using a native PowerShell inline replace command (bypassing Linux 'sed')
                        bat 'powershell -Command "(Get-Content deployment.yaml) -replace \'image: .* university-lms-portal:.*\', \'image: \'+$env:DOCKER_HUB_USER+\'/\'+$env:IMAGE_NAME+\':\'+$env:IMAGE_TAG | Set-Content deployment.yaml"'
                        
                        // Executes the deployment manifests on your local cluster context
                        bat "kubectl apply -f deployment.yaml --kubeconfig=kubeconfig.yaml"
                        bat "kubectl rollout status deployment/university-portal-deployment --kubeconfig=kubeconfig.yaml"
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
                    bat "kubectl rollout undo deployment/university-portal-deployment --kubeconfig=kubeconfig.yaml"
                }
            }
        }
    }
}
