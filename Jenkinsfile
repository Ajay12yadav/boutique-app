def SERVICES = [
  frontend:              [dir: 'src/frontend',              test: 'go'],
  checkoutservice:       [dir: 'src/checkoutservice',       test: 'go'],
  productcatalogservice: [dir: 'src/productcatalogservice', test: 'go'],
  shippingservice:       [dir: 'src/shippingservice',       test: 'go'],
  currencyservice:       [dir: 'src/currencyservice',       test: null],
  paymentservice:        [dir: 'src/paymentservice',        test: null],
  emailservice:          [dir: 'src/emailservice',          test: null],
  recommendationservice: [dir: 'src/recommendationservice', test: null],
  adservice:             [dir: 'src/adservice',             test: null],
  cartservice:           [dir: 'src/cartservice',           ctx: 'src/cartservice/src', test: null],
]

// All services enabled
def ENABLED = [
  'frontend',
  'checkoutservice',
  'productcatalogservice',
  'shippingservice',
  // 'currencyservice',
  // 'paymentservice',
  // 'emailservice',
  // 'recommendationservice',
  // 'adservice',
  // 'cartservice'
]

pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  parameters {
    booleanParam(
      name: 'BUILD_ALL',
      defaultValue: false,
      description: 'Ignore change detection and build every enabled service'
    )
  }

  environment {
    REGISTRY = 'ghcr.io/ajay12yadav'
    GO_IMAGE = 'golang:1.27-alpine'
  }

  stages {

    stage('Detect changes') {
      steps {
        script {

          env.TAG = sh(
            returnStdout: true,
            script: 'git rev-parse --short=7 HEAD'
          ).trim()

          def buildAll = params.BUILD_ALL
          def files = []

          try {
            def base = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT

            if (!base) {
              buildAll = true
            } else {
              files = sh(
                returnStdout: true,
                script: "git diff --name-only ${base} HEAD"
              ).trim().split('\n') as List
            }

          } catch (e) {
            echo "Could not diff against previous build, building everything: ${e}"
            buildAll = true
          }

          // If Jenkinsfile changes, rebuild all services
          if (files.contains('Jenkinsfile')) {
            buildAll = true
          }

          def changed = []

          for (name in ENABLED) {

            def dir = SERVICES[name].dir

            if (
              buildAll ||
              files.any { it.startsWith(dir + '/') }
            ) {
              changed << name
            }
          }

          env.CHANGED = changed.join(',')

          echo "=========================================="
          echo "Image Tag       : ${env.TAG}"
          echo "Build All       : ${buildAll}"
          echo "Services Enabled: ${ENABLED.join(', ')}"
          echo "Services Changed: ${env.CHANGED ?: 'none'}"
          echo "=========================================="
        }
      }
    }

    stage('Secret scan') {
      steps {
        sh '''
          docker run --rm \
            --volumes-from jenkins \
            -w "$WORKSPACE" \
            zricethezav/gitleaks:latest \
            detect \
            --no-git \
            --source . \
            --redact
        '''
      }
    }

    stage('Build changed services') {

      when {
        expression {
          return env.CHANGED?.trim()
        }
      }

      steps {
        script {

          for (name in env.CHANGED.split(',')) {

            def svc   = name
            def cfg   = SERVICES[svc]
            def ctx   = cfg.ctx ?: cfg.dir
            def image = "${env.REGISTRY}/${svc}:${env.TAG}"

            stage(svc) {

              echo "=========================================="
              echo "Building service: ${svc}"
              echo "Directory       : ${cfg.dir}"
              echo "Docker context  : ${ctx}"
              echo "Image           : ${image}"
              echo "=========================================="

              // -----------------------------
              // Unit Tests
              // -----------------------------

              if (cfg.test == 'go') {

                docker.image(env.GO_IMAGE).inside(
                  '-e GOCACHE=/tmp/gocache -e GOPATH=/tmp/go'
                ) {

                  dir(cfg.dir) {
                    sh 'go test ./...'
                  }
                }

              } else {

                echo "${svc}: no tests configured yet"
              }

              // -----------------------------
              // Docker Build
              // -----------------------------

              sh """
                docker build \
                  -t ${image} \
                  ${ctx}
              """

              // -----------------------------
              // Trivy Image Scan
              // -----------------------------

              sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy:latest \
                  image \
                  --severity HIGH,CRITICAL \
                  --ignore-unfixed \
                  --exit-code 1 \
                  ${image}
              """

              // -----------------------------
              // Push to GHCR
              // -----------------------------

              if (env.BRANCH_NAME == 'main') {

                withCredentials([
                  usernamePassword(
                    credentialsId: 'ghcr-creds',
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                  )
                ]) {

                  sh '''
                    echo "$GH_TOKEN" | \
                    docker login ghcr.io \
                      -u "$GH_USER" \
                      --password-stdin
                  '''

                  sh "docker push ${image}"

                  sh '''
                    docker logout ghcr.io
                  '''
                }
              }
            }
          }
        }
      }
    }
  }

  post {

    always {
      sh 'docker image prune -f || true'
      cleanWs()
    }

    success {
      echo "=========================================="
      echo "BUILD SUCCESSFUL"
      echo "Tag: ${env.TAG}"
      echo "=========================================="
    }

    failure {
      echo "=========================================="
      echo "BUILD FAILED"
      echo "URL: ${env.BUILD_URL}"
      echo "=========================================="
    }
  }
}
