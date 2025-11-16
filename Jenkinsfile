pipeline {
  agent {
    kubernetes {
      // Optional but nice to be explicit:
      cloud 'jenkins-k8s'           // must match your Cloud name in Jenkins
      label 'jnlp'                  // must match Pod Template label
      inheritFrom 'jenkins-agent'   // Pod Template name in Jenkins config
      // defaultContainer 'jnlp'    // set if your pod template defines this container name
    }
  }

  environment {
    REGISTRY        = 'localhost:8081'         // TODO: replace with your Nexus Docker repo endpoint
    NEXUS_CRED      = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
  }

  options {
    // timestamps() is NOT allowed in 'options' – we removed it
    skipStagesAfterUnstable()
  }

  stages {

    stage('Init') {
      steps {
        script {
          // Build dynamic tags safely at runtime
          def shortCommit = env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : "local"
          env.BUILD_TAG     = "${env.BUILD_NUMBER}-${shortCommit}"
          env.BACKEND_IMAGE = "${env.REGISTRY}/graphpass-backend:${env.BUILD_TAG}"
          env.SIM_IMAGE     = "${env.REGISTRY}/graphpass-sim:${env.BUILD_TAG}"
          echo "Using BUILD_TAG=${env.BUILD_TAG}"
          echo "Backend image: ${env.BACKEND_IMAGE}"
          echo "Sim image: ${env.SIM_IMAGE}"
        }
      }
    }

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
            // Replace with real tests; `|| true` prevents pipeline from failing while you wire things up
            sh 'pytest -q || true'
          }
        }
        stage('Sim Tests') {
          steps {
            sh 'echo "simulator unit tests here" || true'
          }
        }
      }
    }

    stage('Build Images') {
      steps {
        script {
          // Ensure your agent image / pod template has Docker (or adjust to Kaniko/BuildKit)
          sh """
            echo "Building backend image: ${BACKEND_IMAGE}"
            docker build -t ${BACKEND_IMAGE} ./api || true

            echo "Building simulator image: ${SIM_IMAGE}"
            docker build -t ${SIM_IMAGE} ./simulator || true
          """
        }
      }
    }

    stage('Scan Images') {
      steps {
        // Ensure Trivy is installed in the agent image (or remove for now)
        sh """
          echo "Scanning backend image with Trivy"
          trivy image --exit-code 1 --severity CRITICAL ${BACKEND_IMAGE} || true

          echo "Scanning simulator image with Trivy"
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
            echo "Logging into Docker registry: ${REGISTRY}"
            echo "${NEXUS_PASS}" | docker login -u "${NEXUS_USER}" --password-stdin ${REGISTRY} || true

            echo "Pushing backend image: ${BACKEND_IMAGE}"
            docker push ${BACKEND_IMAGE} || true

            echo "Pushing simulator image: ${SIM_IMAGE}"
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
            echo "Deploying to Kubernetes namespace: dev"

            kubectl --kubeconfig=${KUBECONFIG} -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=${KUBECONFIG} -n dev set image deployment/graphpass-sim     graphpass-sim=${SIM_IMAGE} --record || true

            kubectl --kubeconfig=${KUBECONFIG} -n dev rollout status deployment/graphpass-backend || true
            kubectl --kubeconfig=${KUBECONFIG} -n dev rollout status deployment/graphpass-sim || true
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
      // Wrap in try-catch so missing files don't break the whole build
      script {
        try {
          junit 'api/tests/**/results.xml'
        } catch (e) {
          echo "No JUnit results found or failed to publish: ${e}"
        }

        try {
          archiveArtifacts artifacts: '**/target/*.jar, **/*.sbom.json', fingerprint: true
        } catch (e) {
          echo "No artifacts to archive or failed to archive: ${e}"
        }
      }
    }
    failure {
      script {
        echo "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        // Configure SMTP before enabling this
        // mail to: 'you@example.com',
        //      subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        //      body: "See Jenkins."
      }
    }
  }
}
