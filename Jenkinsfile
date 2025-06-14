pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-2'         
        ECR_REPO = 'nrl-internal'       
        IMAGE_TAG = "${BUILD_NUMBER}"           
        K8S_NAMESPACE = 'default'       
        AWS_ACCOUNT_ID = '090814668573'
        KUBECTL_PATH = "${env.HOME}/bin"
        
    }
    stages {
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
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        sed -i 's|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}|' deployment.yaml
                        git config --global user.email "bhargav.ptd@gmail.com"
                        git config --global user.name "bhargavsiripurapu"
                        git remote set-url origin https://$GITHUB_TOKEN@github.com/bhargavsiripurapu/app-Jenkins-eks-CI-CD.git
                        git add -f deployment.yaml
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
                # Download and install kubectl to a user-writable directory
                mkdir -p ${KUBECTL_PATH}
                curl -o ${KUBECTL_PATH}/kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.6/2024-11-15/bin/linux/amd64/kubectl
                chmod +x ${KUBECTL_PATH}/kubectl

                # Add kubectl to PATH
                echo "export PATH=${KUBECTL_PATH}:\$PATH" >> ~/.bashrc
                export PATH=${KUBECTL_PATH}:\$PATH

                # Verify kubectl
                which kubectl
                kubectl version --client
                """
            }
        }
        stage('Deploy to EKS') {
            steps {
              withCredentials([
                              string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                              string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                            ]) {

                    script {
                        // Export AWS credentials to environment
                        sh '''
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        # Confirm AWS CLI works
                        aws sts get-caller-identity
                         # Update kubeconfig with EKS context
                        aws eks --region ${AWS_REGION} update-kubeconfig --name main-NRL
                        
                        # Check cluster nodes
                        ${KUBECTL_PATH}/kubectl version --client
                        ${KUBECTL_PATH}/kubectl config get-contexts
                        ${KUBECTL_PATH}/kubectl get nodes
                        
                        # Deploy Kubernetes manifests
                        ${KUBECTL_PATH}/kubectl apply -f deployment.yaml
                        ${KUBECTL_PATH}/kubectl apply -f service.yaml
                        ${KUBECTL_PATH}/kubectl apply -f ingress.yaml
                        '''
                    }
                }
            }
        }
        

    }
    
}
