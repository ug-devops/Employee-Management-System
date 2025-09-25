pipeline {
  agent any

  environment {
    AWS_REGION = "us-east-1"                     // change to your region
    AWS_ACCOUNT_ID = "847109903209"              // change to your account id
    ECR_REPO_BACKEND = "ems-backend"
    ECR_REPO_FRONTEND = "ems-frontend"
    BACKEND_TASK_FAMILY = "ems-backend-taskdef"  // family of existing task definition
    FRONTEND_TASK_FAMILY = "ems-frontend-taskdef"
    BACKEND_SERVICE = "ems-backend-service"
    FRONTEND_SERVICE = "ems-frontend-service"
    CLUSTER = "ems-cluster"
    BACKEND_CONTAINER_NAME = "ems-backend"       // container name as defined in task def
    FRONTEND_CONTAINER_NAME = "ems-frontend"    // container name as defined in task def
    IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ?: 'manual'}"
  }

  options {
    ansiColor('xterm')
    timeout(time: 60, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Pre-checks') {
      steps {
        script {
          sh '''
            if ! command -v jq > /dev/null; then
              echo "jq is required but not installed. Install jq on the agent."
              exit 2
            fi
          '''
        }
      }
    }

    stage('ECR Login') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
          sh '''
            echo "Logging into ECR..."
            aws ecr get-login-password --region $AWS_REGION | \
              docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
          '''
        }
      }
    }

    stage('Build & Push Backend') {
      steps {
        dir('ems-backend') {
          withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
            sh '''
              TAG=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_BACKEND:$IMAGE_TAG
              aws ecr describe-repositories --repository-names $ECR_REPO_BACKEND --region $AWS_REGION || \
                aws ecr create-repository --repository-name $ECR_REPO_BACKEND --region $AWS_REGION

              docker build -t $ECR_REPO_BACKEND:$IMAGE_TAG .
              docker tag $ECR_REPO_BACKEND:$IMAGE_TAG $TAG
              docker push $TAG
              echo "Pushed backend image: $TAG"
              echo $TAG > ../backend_image_uri.txt
            '''
          }
        }
      }
    }

    stage('Build & Push Frontend') {
      steps {
        dir('ems-frontend') {
          withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
            sh '''
              TAG=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_FRONTEND:$IMAGE_TAG
              aws ecr describe-repositories --repository-names $ECR_REPO_FRONTEND --region $AWS_REGION || \
                aws ecr create-repository --repository-name $ECR_REPO_FRONTEND --region $AWS_REGION

              docker build -t $ECR_REPO_FRONTEND:$IMAGE_TAG .
              docker tag $ECR_REPO_FRONTEND:$IMAGE_TAG $TAG
              docker push $TAG
              echo "Pushed frontend image: $TAG"
              echo $TAG > ../frontend_image_uri.txt
            '''
          }
        }
      }
    }

    stage('Deploy Backend to ECS') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
          sh '''
            BACKEND_IMAGE=$(cat backend_image_uri.txt)
            echo "Registering backend task definition revision with image $BACKEND_IMAGE"

            aws ecs describe-task-definition --task-definition $BACKEND_TASK_FAMILY --region $AWS_REGION > /tmp/backend_td.json

            jq --arg img "$BACKEND_IMAGE" --arg cname "$BACKEND_CONTAINER_NAME" '
              .taskDefinition
              | { family: .family,
                  networkMode: .networkMode,
                  taskRoleArn: .taskRoleArn,
                  executionRoleArn: .executionRoleArn,
                  containerDefinitions: ( .containerDefinitions
                      | map( if .name == $cname then .image = $img else . end)
                    ),
                  volumes: .volumes,
                  placementConstraints: .placementConstraints,
                  requiresCompatibilities: .requiresCompatibilities,
                  cpu: .cpu,
                  memory: .memory
                }
            ' /tmp/backend_td.json > /tmp/backend-register.json

            REGISTER_OUT=$(aws ecs register-task-definition --cli-input-json file:///tmp/backend-register.json --region $AWS_REGION)
            NEW_TASK_DEF_ARN=$(echo "$REGISTER_OUT" | jq -r '.taskDefinition.taskDefinitionArn')
            echo "New backend task def ARN: $NEW_TASK_DEF_ARN"

            aws ecs update-service --cluster $CLUSTER --service $BACKEND_SERVICE --task-definition $NEW_TASK_DEF_ARN --force-new-deployment --region $AWS_REGION
            echo "Triggered backend service update"
          '''
        }
      }
    }

    stage('Deploy Frontend to ECS') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
          sh '''
            FRONTEND_IMAGE=$(cat frontend_image_uri.txt)
            echo "Registering frontend task definition revision with image $FRONTEND_IMAGE"

            aws ecs describe-task-definition --task-definition $FRONTEND_TASK_FAMILY --region $AWS_REGION > /tmp/frontend_td.json

            jq --arg img "$FRONTEND_IMAGE" --arg cname "$FRONTEND_CONTAINER_NAME" '
              .taskDefinition
              | { family: .family,
                  networkMode: .networkMode,
                  taskRoleArn: .taskRoleArn,
                  executionRoleArn: .executionRoleArn,
                  containerDefinitions: ( .containerDefinitions
                      | map( if .name == $cname then .image = $img else . end)
                    ),
                  volumes: .volumes,
                  placementConstraints: .placementConstraints,
                  requiresCompatibilities: .requiresCompatibilities,
                  cpu: .cpu,
                  memory: .memory
                }
            ' /tmp/frontend_td.json > /tmp/frontend-register.json

            REGISTER_OUT=$(aws ecs register-task-definition --cli-input-json file:///tmp/frontend-register.json --region $AWS_REGION)
            NEW_TASK_DEF_ARN=$(echo "$REGISTER_OUT" | jq -r '.taskDefinition.taskDefinitionArn')
            echo "New frontend task def ARN: $NEW_TASK_DEF_ARN"

            aws ecs update-service --cluster $CLUSTER --service $FRONTEND_SERVICE --task-definition $NEW_TASK_DEF_ARN --force-new-deployment --region $AWS_REGION
            echo "Triggered frontend service update"
          '''
        }
      }
    }

  } // stages

  post {
    always {
      sh 'docker system prune -af || true'
    }
    success {
      echo "Pipeline finished successfully."
    }
    failure {
      echo "Pipeline failed. Check console output."
    }
  }
}

