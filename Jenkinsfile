pipeline {
  agent any
    environment {
      AWS_ACCOUNT_ID="YOUR_ACCOUNT_ID"
      AWS_DEFAULT_REGION="YOU_ACCOUNT_REGION"
      IMAGE_REPO_NAME="YOUR_ECR_REPO_NAME"
      DOCKER_CONTAINER_NAME="YOUR_DOCKER_CONTAINER_NAME"
      GIT_SOURCE_REPO="YOUR_GIT_SOURCE_REPO"
      IMAGE_TAG="latest"
      REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }

    stages {
        
      stage("Cloning Git") {
        steps {
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: '${GIT_SOURCE_REPO}']]])
        }
      }
        
      stage("Logging into AWS ECR") {
        steps {
          script {
            sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
          }
        }
      }
        
      stage("Building image") {
        steps{
          script {
            dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
          }
        }
      }

      stage("Pushing to ECR") {
        steps{
          script {
            sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:$IMAGE_TAG"
            sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
          }
        }
      }
      
      stage("Deleting untagged images from ECR") {
        steps{
          script {
            ECR_IMAGES_TO_DELETE=sh(returnStdout: true, script: 'aws ecr list-images --repository-name $IMAGE_REPO_NAME --filter "tagStatus=UNTAGGED" --query "to_string(imageIds[*])" --output json').trim()
            sh "aws ecr batch-delete-image --repository-name ${IMAGE_REPO_NAME} --image-ids ${ECR_IMAGES_TO_DELETE} || true"
          }
        }
      }
        
      stage('Stop previous containers') {
        steps {
          sh 'docker ps -f name=${DOCKER_CONTAINER_NAME} -q | xargs --no-run-if-empty docker container stop'
          sh 'docker container ls -a -fname=${DOCKER_CONTAINER_NAME} -q | xargs -r docker container rm'
        }
      }
      
      stage('Docker Run') {
        steps{
          script {
            sh 'docker run -d -p 80:80 --rm --name ${DOCKER_CONTAINER_NAME} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}'
          }
        }
      }
      
      stage("Deleting untagged images from Docker") {
        steps{
          script {
            DOCKER_IMAGES_TO_DELETE=sh(returnStdout: true, script: 'docker images -f "dangling=true" -q $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME').trim()
            if (DOCKER_IMAGES_TO_DELETE != "") { 
              sh "docker rmi ${DOCKER_IMAGES_TO_DELETE}"
            }
          }
        }
      }
    }
}
