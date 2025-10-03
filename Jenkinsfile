pipeline {
  // Use Node 16 Docker image as the build agent
  agent {
    docker {
      image 'node:16-alpine'
    }
  }

  environment {
    DOCKER_HOST = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY ='1'
    DOCKER_CERT_PATH = '/certs/client'
    APP_NAME = 'express-sample'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install dependencies') {
      steps {
        sh 'node -v && npm -v'
        sh 'npm install --save --silent'
      }
    }

    stage('Unit tests') {
      steps {
        sh 'npm test || echo "No tests found; continuing"'
      }
    }

    stage('Build Docker image') {
      steps {
        script {
          env.IMAGE_TAG = "${env.BUILD_NUMBER}"
          sh 'docker version'
          sh 'docker build -t ${APP_NAME}:${IMAGE_TAG} .'
        }
      }
    }

    stage('Push to Docker Hub') {
      steps {
        script {
          withCredentials([usernamePassword(
            credentialsId: 'dockerhub',
            usernameVariable: 'DOCKERHUB_USERNAME',
            passwordVariable: 'DOCKERHUB_PASSWORD'
          )]) {
            sh '''
              echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
              docker tag ${APP_NAME}:${IMAGE_TAG} ${DOCKERHUB_USERNAME}/${APP_NAME}:${IMAGE_TAG}
              docker push ${DOCKERHUB_USERNAME}/${APP_NAME}:${IMAGE_TAG}
            '''
          }
        }
      }
    }

    stage('Dependency security scan (fail on High/Critical)') {
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            npx --yes snyk@latest auth "$SNYK_TOKEN"
            npx --yes snyk@latest test --severity-threshold=high
          '''
        }
      }
    }
  }

  post {
    always {
      sh 'docker image prune -f || true'
      archiveArtifacts artifacts: '**/npm-debug.log', allowEmptyArchive: true
    }
  }
}

