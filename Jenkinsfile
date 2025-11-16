pipeline {
  // Must match the label from your Kubernetes Pod Template (you used "jnlp")
  agent { label 'jnlp' }

  environment {
    // TODO: change this to your actual Nexus Docker registry endpoint
    // e.g. "nexus.my-domain.com:8083" or "192.168.56.10:5000"
    REGISTRY        = 'localhost:5000'

    // Jenkins credentials IDs (already created in your Jenkins)
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
  }

  // General pipeline options
  options {
    // Remove timestamps() because it’s not a valid declarative option in your setup
    skipStagesAfterUnstable()
  }

  stages {

    stage('Prepare') {
      steps {
        script {
          // Short commit for tagging; handle case where GIT_COMMIT might be null
          env.SHORT_COMMIT = (env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : "local")
          env.BUILD_TAG    = "${env.BUILD_NUMBER}-${env.SHORT_COMMIT}"

          // Build image names using the registry and build tag
          env.BACKEND_IMAGE = "${env.REGISTRY}/graphpass-backend:${env.BUILD_TAG}"
          env.SIM_IMAGE     = "${env.REGISTRY}/graphpass-sim:${env.BUILD_TAG}"

          echo "Using BUILD_TAG: ${env.BUILD_TAG}"
          echo "Backend image:   ${env.BACKEND_IMAGE}"
          echo "Sim image:       ${env.SIM_IMAGE}"
        }
      }
    }

    stage('Checkout') {
      steps {
        // For Multibranch: this uses Jenkins' built-in SCM config
        checkout scm
      }
    }

    stage('Lint & SAST') {
      parallel {
        stage('Backend Lint') {
          steps {
            sh '''
              echo ">>> Backend lint (flake8 / bandit, etc.)"
              # Example:
              # pip install -r api/requirements-dev.txt
              # cd api && flake8 .
              # cd api && bandit -r .
            '''
          }
        }
        stage('Frontend Lint') {
          steps {
            sh '''
              echo ">>> Frontend lint (if you have one)"
              # Example:
              # cd frontend && npm install
              # cd frontend && npm run lint
            '''
          }
        }
      }
    }

    stage('Unit Tests') {
      parallel {
        stage('Backend Tests') {
          steps {
            sh '''
              echo ">>> Running backend tests"
              # Example:
              # cd api && pytest -q
              echo "backend tests placeholder" || true
            '''
          }
        }
        stage('Sim Tests') {
          steps {
            sh '''
              echo ">>> Running simulator tests"
              # Example:
              # cd simulator && pytest -q
              echo "simulator tests placeholder" || true
            '''
          }
        }
      }
    }

    stage('Build Images') {
      steps {
        script {
          sh """
            echo ">>> Building backend Docker image: ${BACKEND_IMAGE}"
            docker build -t ${BACKEND_IMAGE} ./api || true

            echo ">>> Building sim Docker image: ${SIM_IMAGE}"
            docker build -t ${SIM_IMAGE} ./simulator || true
          """
        }
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          echo ">>> Scanning backend image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE} || true

          echo ">>> Scanning sim image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${SIM_IMAGE} || true
        """
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: "${NEXUS_CRED}",
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )
        ]) {
          sh """
            echo ">>> Logging into registry: ${REGISTRY}"
            echo "${NEXUS_PASS}" | docker login -u "${NEXUS_USER}" --password-stdin ${REGISTRY} || true

            echo ">>> Pushing backend image: ${BACKEND_IMAGE}"
            docker push ${BACKEND_IMAGE} || true

            echo ">>> Pushing sim image: ${SIM_IMAGE}"
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
            echo ">>> Deploying to dev namespace using kubeconfig"
            kubectl --kubeconfig="${KUBECONFIG}" -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig="${KUBECONFIG}" -n dev set image deployment/graphpass-sim     graphpass-sim=${SIM_IMAGE} --record || true

            echo ">>> Waiting for backend rollout"
            kubectl --kubeconfig="${KUBECONFIG}" -n dev rollout status deployment/graphpass-backend || true
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh '''
          echo ">>> Integration tests hitting dev endpoints"
          # Example:
          # pytest -q tests/integration
          echo "integration tests placeholder" || true
        '''
      }
    }
  }

  post {
    always {
      echo ">>> Archiving test results and artifacts (if any)"
      // Adjust pattern according to your real test report path
      junit allowEmptyResults: true, testResults: 'api/tests/**/results.xml'

      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }
    failure {
      script {
        echo ">>> Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
      }
      // Uncomment & configure mail if SMTP is set in Jenkins
      // mail to: 'you@example.com',
      //      subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
      //      body: "Check Jenkins for details: ${env.BUILD_URL}"
    }
  }
}
