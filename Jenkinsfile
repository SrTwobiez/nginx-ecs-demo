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
                script {
                // Variables
                def ecrRepo = "${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com/${params.ECR_REPO}:latest"

                // Login en ECR
                sh "aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com"

                // Construir imagen
                sh "docker build -t ${params.ECR_REPO}:latest ."

                // Tag y push a ECR
                sh "docker tag ${params.ECR_REPO}:latest ${ecrRepo}"
                sh "docker push ${ecrRepo}"
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
