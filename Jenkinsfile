// Jenkinsfile
pipeline {
  // This MUST match the label in your Kubernetes Pod Template
  agent { label 'k8s-agent' }

  environment {
    REGISTRY       = 'localhost:5000'        // TODO: replace with your Nexus Docker registry
    NEXUS_CRED     = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
    // Simple BUILD_TAG so we don't need Groovy methods here
    BUILD_TAG      = "${env.BUILD_NUMBER}"
    BACKEND_IMAGE  = "${REGISTRY}/graphpass-backend:${BUILD_TAG}"
    SIM_IMAGE      = "${REGISTRY}/graphpass-sim:${BUILD_TAG}"
  }

  options {
    // 'timestamps()' is NOT a valid declarative option in your version
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
            // change "|| true" to fail the build once tests are ready
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
            docker build -t ${BACKEND_IMAGE} ./api || true
            docker build -t ${SIM_IMAGE} ./simulator || true
          """
        }
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE} || true
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
            echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY} || true
            docker push ${BACKEND_IMAGE} || true
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
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev rollout status deployment/graphpass-backend
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
      junit 'api/tests/**/results.xml' // adjust paths if needed
      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }
    failure {
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins."
    }
  }
}
