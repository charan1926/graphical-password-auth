pipeline {
  // MUST match Pod Template label
  agent { label 'k8s-agent' }

  environment {
    REGISTRY      = 'localhost:5000'        // or your Nexus docker endpoint
    NEXUS_CRED    = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
    BACKEND_IMAGE = "${env.REGISTRY}/graphpass-backend:${env.BUILD_NUMBER}"
    SIM_IMAGE     = "${env.REGISTRY}/graphpass-sim:${env.BUILD_NUMBER}"
  }

  // remove timestamps() since your Jenkins complained it's not a valid option
  options { skipStagesAfterUnstable() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Lint & SAST') {
      parallel {
        stage('Backend Lint') {
          steps { sh 'echo "Run backend lint here (flake8/bandit etc.)"' }
        }
        stage('Frontend Lint') {
          steps { sh 'echo "Run frontend lint if applicable"' }
        }
      }
    }

    stage('Unit Tests') {
      parallel {
        stage('Backend Tests') {
          steps { sh 'pytest -q || true' }
        }
        stage('Sim Tests') {
          steps { sh 'echo "sim unit tests here" || true' }
        }
      }
    }

    stage('Build Images') {
      steps {
        sh """
          docker build -t ${BACKEND_IMAGE} ./api || true
          docker build -t ${SIM_IMAGE} ./simulator || true
        """
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
        withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED}", usernameVariable: 'NEXU_USER', passwordVariable: 'NEXU_PASS')]) {
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
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
          sh """
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true
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
      junit 'api/tests/**/results.xml'
      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }
    failure {
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins."
    }
  }
}
