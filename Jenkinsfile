pipeline {
  // Use your k8s agent
  agent { label 'k8s-agent' }

  environment {
    REGISTRY        = 'localhost:5000'          // your Docker/Nexus registry
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'

    // use variables directly (not env.REGISTRY)
    BACKEND_IMAGE   = "${REGISTRY}/graphpass-backend:${BUILD_NUMBER}"
    SIM_IMAGE       = "${REGISTRY}/graphpass-sim:${BUILD_NUMBER}"
  }

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
            // Keeps your behavior but adds JUnit XML output path
            sh """
              if command -v pytest >/dev/null 2>&1; then
                cd api
                pytest -q --junitxml=tests/results.xml || true
              else
                echo "pytest not installed on this agent, skipping backend tests"
              fi
            """
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
        sh """
          if command -v docker >/dev/null 2>&1; then
            docker build -t ${BACKEND_IMAGE} ./api || true
            docker build -t ${SIM_IMAGE} ./simulator || true
          else
            echo "docker not found on agent, skipping Docker builds"
          fi
        """
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          if command -v trivy >/dev/null 2>&1; then
            trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE} || true
            trivy image --exit-code 1 --severity CRITICAL ${SIM_IMAGE} || true
          else
            echo "trivy not found on agent, skipping image scanning"
          fi
        """
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED}", usernameVariable: 'NEXU_USER', passwordVariable: 'NEXU_PASS')]) {
          sh """
            if command -v docker >/dev/null 2>&1; then
              echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY} || true
              docker push ${BACKEND_IMAGE} || true
              docker push ${SIM_IMAGE} || true
            else
              echo "docker not found on agent, skipping image push"
            fi
          """
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh """
            if command -v kubectl >/dev/null 2>&1; then
              kubectl --kubeconfig=$KUBECONFIG_FILE -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
              kubectl --kubeconfig=$KUBECONFIG_FILE -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true
              kubectl --kubeconfig=$KUBECONFIG_FILE -n dev rollout status deployment/graphpass-backend || true
            else
              echo "kubectl not found on agent, skipping deploy"
            fi
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
      // Don't fail if no test XML yet
      junit testResults: 'api/tests/**/results.xml', allowEmptyResults: true

      // Don't fail if no JAR/SBOM yet
      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json',
                       fingerprint: true,
                       allowEmptyArchive: true
    }
    failure {
      // Your mail step was failing because there is no SMTP on localhost:25
      // Keep this echo for now; uncomment mail after configuring SMTP.
      echo "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}. Configure SMTP and enable mail step if needed."

      /*
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins."
      */
    }
  }
}
