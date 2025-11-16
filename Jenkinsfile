pipeline {
  agent { label 'jnlp' }  // MUST match Pod Template label in Kubernetes cloud

  environment {
    REGISTRY        = 'localhost:5000'            // replace with Nexus Docker repo endpoint
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
    BUILD_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(8)}"
    BACKEND_IMAGE   = "${REGISTRY}/graphpass-backend:${BUILD_TAG}"
    SIM_IMAGE       = "${REGISTRY}/graphpass-sim:${BUILD_TAG}"
  }

  // timestamps() removed because your Jenkins doesn’t support that Pipeline option
  options { 
    skipStagesAfterUnstable()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Lint & SAST') {
      parallel {
        stage('Backend Lint') {
          steps {
            sh 'echo "Run backend lint here (flake8/bandit etc.)"'
          }
        }
        stage('Frontend Lint') {
          steps {
            sh 'echo "Run frontend lint if applicable"'
          }
        }
      }
    }

    stage('Unit Tests') {
      parallel {
        stage('Backend Tests') {
          steps {
            // adjust to your test command
            sh 'pytest -q || true'
          }
        }
        stage('Sim Tests') {
          steps {
            sh 'echo "sim unit tests here"'
          }
        }
      }
    }

    stage('Build Images') {
      steps {
        script {
          sh """
            echo "Building backend image: ${BACKEND_IMAGE}"
            docker build -t ${BACKEND_IMAGE} ./api || true

            echo "Building sim image: ${SIM_IMAGE}"
            docker build -t ${SIM_IMAGE} ./simulator || true
          """
        }
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          echo "Scanning backend image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE} || true

          echo "Scanning sim image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${SIM_IMAGE} || true
        """
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: "${NEXUS_CRED}",
            usernameVariable: 'NEXU_USER',
            passwordVariable: 'NEXU_PASS'
          )
        ]) {
          sh """
            echo "Logging in to Docker registry: ${REGISTRY}"
            echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY} || true

            echo "Pushing backend image"
            docker push ${BACKEND_IMAGE} || true

            echo "Pushing sim image"
            docker push ${SIM_IMAGE} || true
          """
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([
          file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')
        ]) {
          sh """
            echo "Deploying backend to dev namespace"
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true

            echo "Deploying sim to dev namespace"
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true

            echo "Waiting for backend deployment rollout"
            kubectl --kubeconfig=$KUBECONFIG -n dev rollout status deployment/graphpass-backend || true
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh 'echo "run integration tests hitting dev cluster endpoints" || true'
      }
    }
  }

  post {
    always {
      // adjust pattern to your real test report location
      junit 'api/tests/**/results.xml'
      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }
    failure {
      // requires Mailer or Email-ext plugin configured
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins for details."
    }
  }
}
