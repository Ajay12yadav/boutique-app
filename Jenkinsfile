pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    SERVICE  = 'frontend'
    GO_IMAGE = 'golang:1.27-alpine'
    IMAGE    = 'ghcr.io/ajay12yadav/frontend'
  }

  stages {

    stage('Prepare') {
      steps {
        script {
          env.TAG = sh(
            returnStdout: true,
            script: 'git rev-parse --short=7 HEAD'
          ).trim()
        }

        echo "Building ${env.IMAGE}:${env.TAG}"
      }
    }

    stage('Secret scan') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            zricethezav/gitleaks:latest \
            detect --no-git --source . --redact
        '''
      }
    }

    stage('Unit tests') {
      agent {
        docker {
          image "${GO_IMAGE}"
          reuseNode true
          args '-e GOCACHE=/tmp/gocache -e GOPATH=/tmp/go'
        }
      }

      steps {
        dir('src/frontend') {
          sh 'go test ./...'
        }
      }
    }

    stage('Build image') {
      steps {
        sh 'docker build -t ${IMAGE}:${TAG} src/frontend'
      }
    }

    stage('Image scan') {
      steps {
        sh '''
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest \
            image --severity HIGH,CRITICAL \
            --exit-code 1 \
            ${IMAGE}:${TAG}
        '''
      }
    }

    stage('Push image') {
      when {
        branch 'main'
      }

      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'ghcr-creds',
            usernameVariable: 'GH_USER',
            passwordVariable: 'GH_TOKEN'
          )
        ]) {
          sh '''
            echo "$GH_TOKEN" | docker login ghcr.io \
              -u "$GH_USER" \
              --password-stdin

            docker push ${IMAGE}:${TAG}

            docker logout ghcr.io
          '''
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }

    success {
      echo "OK: ${IMAGE}:${TAG}"
    }

    failure {
      echo "FAILED: ${env.BUILD_URL}"
    }
  }
}