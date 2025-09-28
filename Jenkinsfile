pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-2')
        string(name: 'ECR_REPO', defaultValue: 'nginx-ecs-demo')
        string(name: 'ECS_CLUSTER', defaultValue: 'ecs-lab-cluster')
        string(name: 'ECS_SERVICE', defaultValue: 'nginx-lab-svc')
        string(name: 'TASK_FAMILY', defaultValue: 'nginx-lab-task')
        string(name: 'ACCOUNT_ID', defaultValue: '096863133016')
    }
    stages {
        stage('Checkout') { 
            steps { 
                checkout scm 
            } 
        }
        stage('Build & Push') {
            steps {
                withAWS(credentials: 'aws-lab', region: "${params.AWS_REGION}") {
                    sh '''
                    # Login a ECR
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.$AWS_REGION.amazonaws.com

                    # Construir imagen Docker
                    docker build -t ${ECR_REPO}:latest .

                    # Etiquetar imagen para ECR
                    docker tag ${ECR_REPO}:latest ${ACCOUNT_ID}.dkr.ecr.$AWS_REGION.amazonaws.com/${ECR_REPO}:latest

                    # Subir imagen a ECR
                    docker push ${ACCOUNT_ID}.dkr.ecr.$AWS_REGION.amazonaws.com/${ECR_REPO}:latest
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                withAWS(credentials: 'aws-lab', region: "${params.AWS_REGION}") {
                    sh '''
                    # Forzar nueva deployment en ECS usando la última imagen
                    aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --force-new-deployment
                    '''
                }
            }
        }
    }
}
