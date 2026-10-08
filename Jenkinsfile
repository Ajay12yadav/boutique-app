def SERVICES = [
  frontend:              [dir: 'src/frontend',              test: 'go', libScan: 'enforce'],
  checkoutservice:       [dir: 'src/checkoutservice',       test: 'go', libScan: 'enforce'],
  productcatalogservice: [dir: 'src/productcatalogservice', test: 'go', libScan: 'enforce'],
  shippingservice:       [dir: 'src/shippingservice',       test: 'go', libScan: 'enforce'],
  currencyservice:       [dir: 'src/currencyservice',       test: null, libScan: 'report'],
  paymentservice:        [dir: 'src/paymentservice',        test: null, libScan: 'report'],
  emailservice:          [dir: 'src/emailservice',          test: null, libScan: 'report'],
  recommendationservice: [dir: 'src/recommendationservice', test: null, libScan: 'report'],
  adservice:             [dir: 'src/adservice',             test: null, libScan: 'report'],
  cartservice:           [dir: 'src/cartservice', ctx: 'src/cartservice/src', test: null, libScan: 'report'],
]

// All services enabled
def ENABLED = [
  // 'frontend',
  // 'checkoutservice',
  // 'productcatalogservice',
  // 'shippingservice',
  // 'currencyservice',
  // 'paymentservice',
  // 'emailservice',
  // 'recommendationservice',
  // 'adservice',
  'cartservice'
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

              def libExit = (cfg.libScan == 'enforce') ? 1 : 0

              def trivyBase = "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:latest image --skip-db-update --skip-java-db-update --timeout 10m"

              sh "${trivyBase} --pkg-types os --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ${image}"

              sh "${trivyBase} --pkg-types library --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code ${libExit} ${image}"

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
                  env.PUSHED = (env.PUSHED ? env.PUSHED + ',' : '') + svc

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

    stage('Update GitOps repo') {
      when {
        allOf {
          branch 'main'
          expression { return env.PUSHED?.trim() }
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'gitops-pat',
                                          usernameVariable: 'GH_USER',
                                          passwordVariable: 'GH_TOKEN')]) {
          sh '''
            rm -rf gitops-tmp
            git clone https://${GH_USER}:${GH_TOKEN}@github.com/Ajay12yadav/boutique-gitops.git gitops-tmp
            cd gitops-tmp
            git config user.name "jenkins-ci"
            git config user.email "jenkins@localhost"
          '''
          script {
            for (s in env.PUSHED.split(',')) {
              sh """
                cd gitops-tmp
                sed -i '/name: ${s}\$/,/newTag:/ s/newTag: .*/newTag: "${env.TAG}"/' overlays/dev/kustomization.yaml
              """
            }
          }
          sh '''
            cd gitops-tmp
            git diff --stat
            git commit -am "deploy(dev): ${PUSHED} -> ${TAG} (build ${BUILD_NUMBER})" || echo "nothing to commit"
            for i in 1 2 3; do
              git pull --rebase origin main && git push origin main && break
              sleep 3
            done
          '''
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