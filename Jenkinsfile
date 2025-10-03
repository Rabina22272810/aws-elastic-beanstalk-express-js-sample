pipeline {
  agent any 

  environment {
    
    REGISTRY   = 'docker.io/rabinashrestha'
    IMAGE_NAME = 'aws-eb-express'
    IMAGE_TAG  = "${env.BUILD_NUMBER}"

    // Talk to your DinD from Task 2
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH  = '/certs/client'
  }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install dependencies (Node 16)') {
      steps {
        sh '''
          docker run --rm \
            --network bridge \
            -u 0:0 \
            -v "$WORKSPACE":/workspace \
            -v "$WORKSPACE/.npm":/root/.npm \
            -w /workspace \
            node:16 sh -lc '
              node -v
              npm -v
              npm install --save
            '
        '''
      }
    }

    stage('Unit tests (Node 16)') {
      steps {
        sh '''
          docker run --rm \
            --network bridge \
            -u 0:0 \
            -v "$WORKSPACE":/workspace \
            -v "$WORKSPACE/.npm":/root/.npm \
            -w /workspace \
            node:16 sh -lc "
              npm test --if-present
            "
        '''
      }
    }

    stage('Dependency security scan (Snyk)') {
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            docker run --rm \
              --network bridge \
              -u 0:0 \
              -v "$WORKSPACE":/workspace \
              -w /workspace \
              node:16 sh -lc "
                npm install --save
                npm install -g snyk
                snyk auth $SNYK_TOKEN
                snyk test --severity-threshold=high
              "
          '''
        }
      }
    }

    stage('Build Docker image') {
      environment { DOCKER_BUILDKIT = '1' }
      steps {
        sh '''
          docker build -t $REGISTRY/$IMAGE_NAME:$IMAGE_TAG .
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
            docker tag  $REGISTRY/$IMAGE_NAME:$IMAGE_TAG $REGISTRY/$IMAGE_NAME:latest
            docker push $REGISTRY/$IMAGE_NAME:latest
          '''
        }
      }
    }
  }

  post {
    always {
      junit testResults: 'reports/junit/*.xml', allowEmptyResults: true
      archiveArtifacts artifacts: 'Dockerfile', allowEmptyArchive: true
    }
  }
}
