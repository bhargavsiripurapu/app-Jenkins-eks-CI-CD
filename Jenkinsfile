pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-2'         
        ECR_REPO = 'nrl-internal'       
        IMAGE_TAG = 'latest'            
        K8S_NAMESPACE = 'default'       
        AWS_ACCOUNT_ID = '090814668573'
        
    }
    stages {
        stage('Set AWS Credentials') {
            steps {
                script {
                    // Retrieve AWS credentials from Jenkins credentials store
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', 
                                                       usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                       passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        // Set environment variables for AWS CLI
                        sh 'aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID'
                        sh 'aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY'
                        sh 'aws configure set region ap-south-2'  // Specify your region
                        sh 'aws s3 ls'
                    }
                }
            }
        }
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/bhargavsiripurapu/app-Jenkins-eks-CI-CD.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}" , "-f login-app/Dockerfile .")
                }
            }
        }
        stage('Push to ECR') {
            steps {
                script {
                   sh """
                   aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                   docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                   """
                 
                }
            }
        }
        stage('Update Deployment File') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token-id', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        sed -i 's|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}|' deployment.yaml
                        git config --global user.email "bhargav.ptd@gmail.com"
                        git config --global user.name "bhargavsiripurapu"
                        git remote set-url origin https://$GITHUB_TOKEN@github.com/bhargavsiripurapu/app-Jenkins-eks-CI-CD.git
                        git add deployment.yaml
                        git commit -m "Updated image to ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
                        git push origin main
                        """
                    }
                }
            }
        }
        stage('Install kubectl') {
            steps {
                sh """
                # Download and install kubectl
                curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.6/2024-11-15/bin/linux/amd64/kubectl
                chmod +x ./kubectl
        
                # Move kubectl to /usr/local/bin
                sudo mv ./kubectl /usr/local/bin/kubectl
                
                # Update the PATH for the current session
                export PATH=\$PATH:/usr/local/bin
                
                # Verify kubectl installation
                echo "kubectl installed in PATH: \$PATH"
                which kubectl
                kubectl version --client
                """
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'k8s-config-base64', variable: 'KUBECONFIG_BASE64')]) {
                        writeFile file: 'kubeconfig.base64', text: "${KUBECONFIG_BASE64}"
                        
                        // Decode the base64 kubeconfig to the actual kubeconfig file
                        sh """
                        base64 -d kubeconfig.base64 > kubeconfig
                        
                        # Verify the content of the kubeconfig to ensure the correct context is in place
                        cat kubeconfig
                        
                        # Export the KUBECONFIG environment variable
                        export KUBECONFIG=\$(pwd)/kubeconfig
                        
                        # Use AWS CLI to configure kubectl with EKS cluster credentials
                        aws eks --region ${AWS_REGION} update-kubeconfig --name prod-nrl-nrl_internal
                        
                        # Now kubectl can use the proper context set by EKS
                        kubectl version --client
                        kubectl get nodes  # Sample command to check cluster access
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml
                        kubectl apply -f ingress.yaml
                        """
                    }
                }
            }
        }

    }
    
}
