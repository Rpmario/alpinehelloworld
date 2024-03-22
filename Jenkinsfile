pipeline {
  environment {
    ID_DOCKER = "${ID_DOCKER_PARAMS}" // This should be set in Jenkins job parameters
    IMAGE_NAME = "alpinehelloworld"
    IMAGE_TAG = "latest"
    PORT_EXPOSED = "54993" // Host port mapped to the container's exposed port
    STAGING = "${ID_DOCKER}-staging"
    PRODUCTION = "${ID_DOCKER}-production"
  }
  agent any
  stages {
    stage('Build image') {
      steps {
        bat 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
      }
    }
    stage('Run container based on builded image') {
      steps {
        bat '''
           echo "Clean Environment"
           docker rm -f $IMAGE_NAME || echo "container does not exist"
           docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:5000 -e PORT=5000 ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
           timeout 5
        '''
      }
    }
    stage('Test image') {
      steps {
        bat '''
            curl http://localhost:${PORT_EXPOSED} | findstr /C:"Hello world!"
        '''
      }
    }
    stage('Clean Container') {
      steps {
        bat '''
          docker stop $IMAGE_NAME
          docker rm $IMAGE_NAME
        '''
      }
    }
    stage('Login and Push Image on docker hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-login', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
          bat 'echo $DOCKERHUB_PASS | docker login --username $DOCKERHUB_USER --password-stdin'
          bat 'docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}'
        }
      }
    }
    stage('Push image in staging and deploy it') {
      when {
        expression { GIT_BRANCH == 'origin/master' }
      }
      steps {
        withCredentials([string(credentialsId: 'heroku_api_key', variable: 'HEROKU_API_KEY')]) {
          bat 'heroku container:login'
          bat 'heroku create $STAGING || echo "project already exist"'
          bat 'heroku container:push web -a $STAGING'
          bat 'heroku container:release web -a $STAGING'
        }
      }
    }
    stage('Push image in production and deploy it') {
      when {
        expression { GIT_BRANCH == 'origin/production' }
      }
      steps {
        withCredentials([string(credentialsId: 'heroku_api_key', variable: 'HEROKU_API_KEY')]) {
          bat 'heroku container:login'
          bat 'heroku create $PRODUCTION || echo "project already exist"'
          bat 'heroku container:push web -a $PRODUCTION'
          bat 'heroku container:release web -a $PRODUCTION'
        }
      }
    }
  }
}
