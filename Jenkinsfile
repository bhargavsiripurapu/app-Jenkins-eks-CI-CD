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
                    sh """
                    sed -i 's|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}|' k8s/deployment.yaml
                    git config user.email "bhargav.ptd@gmail.com"
                    git config user.name "bhargavsiripurapu"
                    git add k8s/deployment.yaml
                    git commit -m "Updated image to ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
                    git push origin main
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'k8s-credentials-id') {
                        sh """
                        kubectl config use-context arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/prod-nrl-nrl_internal
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl apply -f k8s/ingress.yaml
                        """
                    }
                }
            }
        }
    }
    
}
