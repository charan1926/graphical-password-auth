pipeline {
  agent {
    kubernetes {
      // This MUST match the Cloud name in "Manage Jenkins → Configure System → Cloud"
      cloud 'jenkins-k8s'

      // This MUST match the "Labels" field in your Pod Template
      label 'jnlp'

      // Container name that runs the agent (in Pod template UI, the container should also be called 'jnlp')
      defaultContainer 'jnlp'

      // If your pod template is named "jenkins-agent", we can inherit it (optional but nice)
      // comment this line if your Jenkins version/plugin complains
      inheritFrom 'jenkins-agent'
    }
  }

  environment {
    REGISTRY        = 'localhost:5000'          // TODO: replace with your Nexus Docker repo endpoint
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
    BUILD_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(8)}"
    BACKEND_IMAGE   = "${REGISTRY}/graphpass-backend:${BUILD_TAG}"
    SIM_IMAGE       = "${REGISTRY}/graphpass-sim:${BUILD_TAG}"
  }

  // 'timestamps()' was causing an option error on your Jenkins, so we only keep skipStagesAfterUnstable
  options {
    skipStagesAfterUnstable()
  }

  stages {

    stage('Checkout') {
      steps {
        // In a multibranch job, this uses the Jenkinsfile's repo/branch
        checkout scm
      }
    }

    stage('Lint & SAST') {
      parallel {
        stage('Backend Lint') {
          steps {
            sh '''
              echo "Run backend lint here (flake8/bandit etc.)"
              # Example:
              # pip install -r api/requirements-dev.txt
              # flake8 api || true
            '''
          }
        }

        stage('Frontend Lint') {
          steps {
            sh '''
              echo "Run frontend lint if applicable"
              # Example:
              # cd frontend && npm install && npm run lint || true
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
              echo "Running backend unit tests"
              # Example:
              # cd api
              # pytest -q || true
            '''
          }
        }

        stage('Sim Tests') {
          steps {
            sh '''
              echo "Sim unit tests here"
              # Example:
              # cd simulator
              # pytest -q || true
            '''
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
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )
        ]) {
          sh """
            echo "Logging in to registry: ${REGISTRY}"
            echo "$NEXUS_PASS" | docker login -u "$NEXUS_USER" --password-stdin ${REGISTRY} || true

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
            echo "Deploying to dev namespace using kubeconfig"
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true

            echo "Waiting for backend deployment rollout"
            kubectl --kubeconfig=$KUBECONFIG -n dev rollout status deployment/graphpass-backend || true
          """
        }
      }
    }

    stage('Integration Tests') {
      steps {
        sh '''
          echo "Run integration tests hitting dev cluster endpoints here"
          # e.g., curl or pytest hitting k8s services
        '''
      }
    }
  }

  post {
    always {
      // Adjust paths to your actual reports; currently just examples
      junit allowEmptyResults: true, testResults: 'api/tests/**/results.xml'
      archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
    }

    failure {
      mail to: 'you@example.com',
           subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
           body: "See Jenkins for details: ${env.BUILD_URL}"
    }
  }
}
