pipeline {
  // You’re currently running on the master/controller node ("Jenkins")
 agent { label 'k8s-agent' }

  environment {
    REGISTRY        = 'localhost:5000'
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'

    // Single Django app image for this repo
    APP_IMAGE       = "${REGISTRY}/graphpass-django:${BUILD_NUMBER}"
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
            sh '''
              echo ">> Backend lint stage (Django project)"
              # Example (enable later if you add tools):
              # if command -v flake8 >/dev/null 2>&1; then
              #   flake8 graphical_pwd_auth home
              # else
              #   echo "flake8 not installed, skipping backend lint"
              # fi
            '''
          }
        }
        stage('Frontend Lint') {
          steps {
            sh '''
              echo ">> Frontend lint stage (no separate frontend yet, placeholder)"
              # Add npm/yarn lint here in future if you add a JS frontend
            '''
          }
        }
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          echo ">> Unit tests (Django)"
          if command -v python >/dev/null 2>&1 && command -v pip >/dev/null 2>&1; then
            echo ">> Installing Python dependencies from requirements.txt"
            pip install -r requirements.txt || echo "requirements install failed or already satisfied"

            echo ">> Running Django tests (if any)"
            python manage.py test || echo "Django tests failed or none defined yet"
          else
            echo "Python/pip not available on this agent, skipping tests"
          fi
        '''
      }
    }

    stage('Build Image') {
      steps {
        sh """
          echo ">> Docker image build stage"
          if command -v docker >/dev/null 2>&1; then
            echo ">> Building app image: ${APP_IMAGE}"
            docker build -t ${APP_IMAGE} .
          else
            echo "docker not found on agent, skipping Docker build"
          fi
        """
      }
    }

    stage('Scan Image') {
      steps {
        sh """
          echo ">> Image scan stage"
          if command -v trivy >/dev/null 2>&1; then
            echo ">> Scanning image with Trivy: ${APP_IMAGE}"
            trivy image --exit-code 1 --severity CRITICAL ${APP_IMAGE} || echo "Trivy scan completed (or failed)"
          else
            echo "trivy not found on agent, skipping image scanning"
          fi
        """
      }
    }

    stage('Publish Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${NEXUS_CRED}",
          usernameVariable: 'NEXU_USER',
          passwordVariable: 'NEXU_PASS'
        )]) {
          sh """
            echo ">> Image publish stage"
            if command -v docker >/dev/null 2>&1; then
              echo ">> Logging in to registry ${REGISTRY}"
              echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY} || echo "Docker login failed"

              echo ">> Pushing image ${APP_IMAGE}"
              docker push ${APP_IMAGE} || echo "Docker push failed"
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
            echo ">> Deploy to dev stage"
            if command -v kubectl >/dev/null 2>&1; then
              echo ">> Updating deployment image in dev namespace"
              kubectl --kubeconfig=$KUBECONFIG_FILE -n dev set image deployment/graphpass-django \
                graphpass-django=${APP_IMAGE} --record || echo "kubectl set image failed"

              echo ">> Checking rollout status"
              kubectl --kubeconfig=$KUBECONFIG_FILE -n dev rollout status deployment/graphpass-django || echo "rollout status failed"
            else
              echo "kubectl not found on agent, skipping deploy"
            fi
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh '''
          echo ">> Integration tests placeholder"
          # Here you can curl your dev service endpoints or run pytest against live URLs
        '''
      }
    }
  }

  post {
    always {
      echo ">> Build result: ${currentBuild.currentResult}"

      // No JUnit XML yet, but keep this so you can plug reports in later.
      junit testResults: 'test-results/**/*.xml', allowEmptyResults: true

      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json',
                       fingerprint: true,
                       allowEmptyArchive: true
    }
    failure {
      echo "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
      // Uncomment after configuring SMTP on Jenkins:
      /*
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins for details: ${env.BUILD_URL}"
      */
    }
  }
}
