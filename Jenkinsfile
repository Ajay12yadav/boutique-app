// pipeline {
//   agent any

//   options {
//     timestamps()
//     timeout(time: 30, unit: 'MINUTES')
//     disableConcurrentBuilds()
//     buildDiscarder(logRotator(numToKeepStr: '10'))
//   }

//   environment {
//     SERVICE  = 'frontend'
//     GO_IMAGE = 'golang:1.27-alpine'
//     IMAGE    = 'ghcr.io/ajay12yadav/frontend'
//   }

//   stages {

//     stage('Prepare') {
//       steps {
//         script {
//           env.TAG = sh(
//             returnStdout: true,
//             script: 'git rev-parse --short=7 HEAD'
//           ).trim()
//         }

//         echo "Building ${env.IMAGE}:${env.TAG}"
//       }
//     }

//     stage('Secret scan') {
//       steps {
//         sh '''
//           docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
//             zricethezav/gitleaks:latest \
//             detect --no-git --source . --redact
//         '''
//       }
//     }

//     stage('Unit tests') {
//       agent {
//         docker {
//           image "${GO_IMAGE}"
//           reuseNode true
//           args '-e GOCACHE=/tmp/gocache -e GOPATH=/tmp/go'
//         }
//       }

//       steps {
//         dir('src/frontend') {
//           sh 'go test ./...'
//         }
//       }
//     }

//     stage('Build image') {
//       steps {
//         sh 'docker build -t ${IMAGE}:${TAG} src/frontend'
//       }
//     }

//     stage('Image scan') {
//       steps {
//         sh '''
//           docker run --rm \
//             -v /var/run/docker.sock:/var/run/docker.sock \
//             aquasec/trivy:latest \
//             image --severity HIGH,CRITICAL \
//             --exit-code 1 \
//             ${IMAGE}:${TAG}
//         '''
//       }
//     }

//     stage('Push image') {
//       when {
//         branch 'main'
//       }

//       steps {
//         withCredentials([
//           usernamePassword(
//             credentialsId: 'ghcr-creds',
//             usernameVariable: 'GH_USER',
//             passwordVariable: 'GH_TOKEN'
//           )
//         ]) {
//           sh '''
//             echo "$GH_TOKEN" | docker login ghcr.io \
//               -u "$GH_USER" \
//               --password-stdin

//             docker push ${IMAGE}:${TAG}

//             docker logout ghcr.io
//           '''
//         }
//       }
//     }
//   }

//   post {
//     always {
//       cleanWs()
//     }

//     success {
//       echo "OK: ${IMAGE}:${TAG}"
//     }

//     failure {
//       echo "FAILED: ${env.BUILD_URL}"
//     }
//   }
// }
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
  cartservice:           [dir: 'src/cartservice', ctx: 'src/cartservice/src', test: null],
]

// Rollout waves: start small, add names as each wave goes green
def ENABLED = ['frontend', 'checkoutservice', 'productcatalogservice', 'shippingservice']

pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  parameters {
    booleanParam(name: 'BUILD_ALL', defaultValue: false,
                 description: 'Ignore change detection and build every enabled service')
  }

  environment {
    REGISTRY = 'ghcr.io/ajay12yadav'
    GO_IMAGE = 'golang:1.27-alpine'
  }

  stages {
    stage('Detect changes') {
      steps {
        script {
          env.TAG = sh(returnStdout: true, script: 'git rev-parse --short=7 HEAD').trim()

          def buildAll = params.BUILD_ALL
          def files = []
          try {
            def base = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
            if (!base) {
              buildAll = true
            } else {
              files = sh(returnStdout: true,
                         script: "git diff --name-only ${base} HEAD").trim().split('\n') as List
            }
          } catch (e) {
            echo "Could not diff against previous build, building everything: ${e}"
            buildAll = true
          }

          if (files.contains('Jenkinsfile')) { buildAll = true }

          def changed = []
          for (name in ENABLED) {
            def dir = SERVICES[name].dir
            if (buildAll || files.any { it.startsWith(dir + '/') }) {
              changed << name
            }
          }
          env.CHANGED = changed.join(',')
          echo "Tag: ${env.TAG}"
          echo "Services to build: ${env.CHANGED ?: 'none'}"
        }
      }
    }

    stage('Secret scan') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            zricethezav/gitleaks:latest detect --no-git --source . --redact
        '''
      }
    }

    stage('Build changed services') {
      when { expression { env.CHANGED } }
      steps {
        script {
          for (name in env.CHANGED.split(',')) {
            def svc   = name
            def cfg   = SERVICES[svc]
            def ctx   = cfg.ctx ?: cfg.dir
            def image = "${env.REGISTRY}/${svc}:${env.TAG}"

            stage(svc) {
              if (cfg.test == 'go') {
                docker.image(env.GO_IMAGE).inside('-e GOCACHE=/tmp/gocache -e GOPATH=/tmp/go') {
                  dir(cfg.dir) { sh 'go test ./...' }
                }
              } else {
                echo "${svc}: no tests configured yet"
              }

              sh "docker build -t ${image} ${ctx}"

              sh """docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy:latest image --severity HIGH,CRITICAL \
                    --ignore-unfixed --exit-code 1 ${image}"""

              if (env.BRANCH_NAME == 'main') {
                withCredentials([usernamePassword(credentialsId: 'ghcr-creds',
                                                  usernameVariable: 'GH_USER',
                                                  passwordVariable: 'GH_TOKEN')]) {
                  sh '''
                    echo "$GH_TOKEN" | docker login ghcr.io -u "$GH_USER" --password-stdin
                  '''
                  sh "docker push ${image}"
                  sh 'docker logout ghcr.io'
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
    failure { echo "FAILED: ${env.BUILD_URL}" }
  }
}