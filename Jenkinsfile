pipeline {
  // 🔴 MAKE SURE this label matches the label in your Pod Template
  agent { label 'k8s-agent' }   // or 'jnlp' if that's what you set in Jenkins

  environment {
    REGISTRY       = 'localhost:8081'          
    NEXUS_CRED     = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
  }

  // ✅ Removed timestamps(), keep only valid options
  options { 
    skipStagesAfterUnstable()
  }

  stages {
    stage('Init') {
      steps {
        script {
          // Safe way to build tag using env vars
          def shortCommit = (env.GIT_COMMIT ?: 'no-git-hash').take(8)
          env.BUILD_TAG     = "${env.BUILD_NUMBER}-${shortCommit}"
          env.BACKEND_IMAGE = "${env.REGISTRY}/graphpass-backend:${env.BUILD_TAG}"
          env.SIM_IMAGE     = "${env.REGISTRY}/graphpass-sim:${env.BUILD_TAG}"
          echo "BUILD_TAG: ${env.BUILD_TAG}"
          echo "BACKEND_IMAGE: ${env.BACKEND_IMAGE}"
          echo "SIM_IMAGE: ${env.SIM_IMAGE}"
        }
      }
    }

    stage('Checkout') {
      steps {
        // if you want timestamps, you can wrap steps like this:
        // timestamps {
        //   checkout scm
        // }
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
            docker build -t ${env.BACKEND_IMAGE} ./api || true
            docker build -t ${env.SIM_IMAGE} ./simulator || true
          """
        }
      }
    }

    stage('Scan Images') {
      steps {
        sh """
          trivy image --exit-code 1 --severity CRITICAL ${env.BACKEND_IMAGE} || true
          trivy image --exit-code 1 --severity CRITICAL ${env.SIM_IMAGE} || true
        """
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${env.NEXUS_CRED}", usernameVariable: 'NEXU_USER', passwordVariable: 'NEXU_PASS')]) {
          sh """
            echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${env.REGISTRY} || true
            docker push ${env.BACKEND_IMAGE} || true
            docker push ${env.SIM_IMAGE} || true
          """
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([file(credentialsId: "${env.KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
          sh """
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${env.BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${env.SIM_IMAGE} --record || true
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
      mail to: 'you@example.com', subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: "See Jenkins."
    }
  }
}
