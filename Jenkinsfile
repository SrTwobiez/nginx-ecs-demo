pipeline {
    agent any

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-2')
        string(name: 'ECR_REPO', defaultValue: 'nginx-ecs-demo')
        string(name: 'ECS_CLUSTER', defaultValue: 'ecs-lab-cluster')
        string(name: 'ECS_SERVICE', defaultValue: 'nginx-lab-svc')
        string(name: 'TASK_FAMILY', defaultValue: 'nginx-lab-task')
        string(name: 'ACCOUNT_ID', defaultValue: '096863133016') // <== MODIFICADO PARA TI
    }

    environment {
        ECR_URL = "${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com"
        IMAGE_TAG = "latest"
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
                    // Construye la imagen de Docker para la aplicación Nginx
                    docker.build("${ECR_URL}/${params.ECR_REPO}:${IMAGE_TAG}")
                    
                    // Se autentica con AWS ECR y sube la imagen
                    withAWS(credentials: 'aws-lab', region: params.AWS_REGION) {
                        sh "aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}"
                        sh "docker push ${ECR_URL}/${params.ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-lab', region: params.AWS_REGION) {
                        // 1. Obtiene la definición de la tarea actual
                        def taskDef = sh(
                            script: "aws ecs describe-task-definition --task-definition ${params.TASK_FAMILY} --region ${params.AWS_REGION} --output json",
                            returnStdout: true
                        ).trim()

                        // 2. Filtra la definición para quedarnos solo con los campos necesarios para una nueva revisión
                        def filteredDef = sh(
                             script: """echo '${taskDef}' | jq '.taskDefinition | {family, containerDefinitions, executionRoleArn, taskRoleArn, networkMode, volumes, placementConstraints, requiresCompatibilities, cpu, memory, pidMode, ipcMode, proxyConfiguration, inferenceAccelerators, ephemeralStorage, runtimePlatform}' | jq 'with_entries(select(.value != null))'""",
                            returnStdout: true
                        ).trim()

                        // 3. Actualiza la definición del contenedor con la URL de la nueva imagen en ECR
                        def updatedDef = sh(
                             script: "echo '${filteredDef}' | jq '.containerDefinitions[0].image = \"${ECR_URL}/${params.ECR_REPO}:${IMAGE_TAG}\"'",
                            returnStdout: true
                        ).trim()

                        sh "echo '${updatedDef}' > task-def.json"
                        
                        // 4. Registra una nueva revisión de la definición de tarea
                        def newTaskDef = sh(
                            script: "aws ecs register-task-definition --region ${params.AWS_REGION} --cli-input-json file://task-def.json --output json",
                            returnStdout: true
                        ).trim()

                        def newArn = sh(
                            script: "echo '${newTaskDef}' | jq -r '.taskDefinition.taskDefinitionArn'",
                            returnStdout: true
                        ).trim()

                        // 5. Actualiza el servicio de ECS para que use la nueva revisión de la tarea
                        sh "aws ecs update-service --cluster ${params.ECS_CLUSTER} --service ${params.ECS_SERVICE} --task-definition ${newArn} --region ${params.AWS_REGION}"
                    }
                }
            }
        }
    }
}