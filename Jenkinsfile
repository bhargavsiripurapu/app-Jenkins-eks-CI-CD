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
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                mkdir -p /var/lib/jenkins/bin
                echo "your_password_here" | sudo -S mv ./kubectl /usr/local/bin/kubectl
                export PATH=/var/lib/jenkins/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
                """
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'k8s-config-base64', variable: 'KUBECONFIG_BASE64')]) {
                        writeFile file: 'kubeconfig.base64', text: "${KUBECONFIG_BASE64}"
                        sh """
                        base64 -d kubeconfig.base64 > kubeconfig
                        kubectl --kubeconfig=kubeconfig config use-context arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/prod-nrl-nrl_internal
                        kubectl --kubeconfig=kubeconfig apply -f deployment.yaml
                        kubectl --kubeconfig=kubeconfig apply -f service.yaml
                        kubectl --kubeconfig=kubeconfig apply -f ingress.yaml
                        """
                    }
                }
            }
        }
    }
    
}
