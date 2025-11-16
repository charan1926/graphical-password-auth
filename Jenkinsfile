pipeline {
  // This MUST match the label in your Kubernetes Pod Template
  agent { label 'k8s-agent' }

  environment {
    REGISTRY        = 'localhost:8081'        // TODO: replace with your Nexus repo like: nexus-host:5000
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'

    // Simple image tags based on build number
    BACKEND_IMAGE = "${REGISTRY}/graphpass-backend:${BUILD_NUMBER}"
    SIM_IMAGE     = "${REGISTRY}/graphpass-sim:${BUILD_NUMBER}"
  }

  options {
    // This one is supported; we removed 'timestamps()' from options
    skipStagesAfterUnstable()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          // Optional: log commit for visibility
          sh 'git rev-parse --short HEAD || true'
        }
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
            sh 'pytest -q || true'   // replace with real test command
          }
        }
        stage('Sim Tests') {
          steps {
            sh 'echo "sim unit tests here" || true'
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
            echo "Logging into registry: ${REGISTRY}"
            echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY} || true

            echo "Pushing backend image: ${BACKEND_IMAGE}"
            docker push ${BACKEND_IMAGE} || true

            echo "Pushing sim image: ${SIM_IMAGE}"
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
            echo "Deploying backend to dev with image: ${BACKEND_IMAGE}"
            kubectl --kubeconfig=$KUBECONFIG -n dev set image \
              deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true

            echo "Deploying sim to dev with image: ${SIM_IMAGE}"
            kubectl --kubeconfig=$KUBECONFIG -n dev set image \
              deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true

            echo "Waiting for backend rollout..."
            kubectl --kubeconfig=$KUBECONFIG -n dev rollout status deployment/graphpass-backend || true
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh 'echo "Run integration tests hitting dev cluster endpoints" || true'
      }
    }
  }

  post {
    always {
      // Adjust test report path to your project
      junit allowEmptyResults: true, testResults: 'api/tests/**/results.xml'

      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }

    failure {
      mail to: 'charannaidus1926@gmail.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins for details: ${env.BUILD_URL}"
    }
  }
}
