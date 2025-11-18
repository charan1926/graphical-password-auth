pipeline {
  // Change to 'any' if you haven't created a k8s agent yet
  agent { label 'k8s-agent' }

  environment {
    REGISTRY        = 'localhost:5000'          // your Docker registry (or Nexus)
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'

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
            sh '''
              cd api
              echo ">> Running backend lint (flake8/bandit)..."
              # pip install -r requirements.txt
              # flake8 .
              # bandit -r .
            '''
          }
        }
        stage('Frontend Lint') {
          steps {
            sh '''
              echo ">> Running frontend lint..."
              # put npm/yarn lint commands here if you have a frontend
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
              cd api
              echo ">> Running backend unit tests..."
              # pip install -r requirements.txt
              pytest -q --junitxml=test-results/pytest-results.xml
            '''
          }
        }
        stage('Sim Tests') {
          steps {
            sh '''
              echo ">> Running simulator tests..."
              # cd simulator
              # pytest -q --junitxml=test-results/pytest-results.xml
              echo "sim unit tests here"
            '''
          }
        }
      }
    }

    stage('Build Images') {
      steps {
        sh """
          echo ">> Building backend image: ${BACKEND_IMAGE}"
          docker build -t ${BACKEND_IMAGE} ./api

          echo ">> Building simulator image: ${SIM_IMAGE}"
          docker build -t ${SIM_IMAGE} ./simulator
        """
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          echo ">> Scanning backend image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE}

          echo '>> Scanning simulator image with Trivy'
          trivy image --exit-code 1 --severity CRITICAL ${SIM_IMAGE}
        """
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${NEXUS_CRED}",
          usernameVariable: 'NEXU_USER',
          passwordVariable: 'NEXU_PASS'
        )]) {
          sh """
            echo ">> Logging into registry ${REGISTRY}"
            echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY}

            echo ">> Pushing backend image"
            docker push ${BACKEND_IMAGE}

            echo ">> Pushing simulator image"
            docker push ${SIM_IMAGE}
          """
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh """
            echo ">> Deploying to dev namespace with kubectl"

            kubectl --kubeconfig=$KUBECONFIG_FILE -n dev set image deployment/graphpass-backend \
              graphpass-backend=${BACKEND_IMAGE} --record

            kubectl --kubeconfig=$KUBECONFIG_FILE -n dev set image deployment/graphpass-sim \
              graphpass-sim=${SIM_IMAGE} --record

            echo ">> Waiting for backend rollout to finish"
            kubectl --kubeconfig=$KUBECONFIG_FILE -n dev rollout status deployment/graphpass-backend
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh '''
          echo ">> Running integration tests against Dev cluster..."
          # curl or pytest hitting your dev endpoints here
        '''
      }
    }
  }

  post {
    always {
      echo "Build result: ${currentBuild.currentResult}"

      // Collect test reports (backend path)
      junit testResults: 'api/test-results/**/*.xml', allowEmptyResults: true

      // Optional: if you add reports for sim, adjust pattern
      // junit testResults: 'simulator/test-results/**/*.xml', allowEmptyResults: true

      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json',
                       fingerprint: true,
                       allowEmptyArchive: true
    }
    failure {
      echo "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
      // Uncomment after configuring SMTP in Jenkins:
      /*
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins for details: ${env.BUILD_URL}"
      */
    }
  }
}
