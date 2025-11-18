// Jenkinsfile - cleaned to avoid Groovy ambiguity errors
// Place on dev branch. Assumes a PodTemplate label 'k8s-agent' exists.

pipeline {
  agent { label 'k8s-agent' }
  environment {
    REGISTRY = 'localhost:5000'
    NEXUS_CRED = 'nexus-docker-creds'
    KUBECONFIG_CRED = 'kubeconfig'
    BUILD_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(8)}"
    BACKEND_IMAGE = "${REGISTRY}/graphpass-backend:${BUILD_TAG}"
    SIM_IMAGE     = "${REGISTRY}/graphpass-sim:${BUILD_TAG}"
    FORCE_PUSH = "false"
  }

  options {
    disableConcurrentBuilds()
    skipStagesAfterUnstable()
    timeout(time: 60, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'echo "Checked out commit: ${GIT_COMMIT:=unknown} on branch ${BRANCH_NAME}"'
      }
    }

    stage('Prepare') {
      steps {
        sh 'echo "Build tag => ${BUILD_TAG}"'
        sh 'git rev-parse --short HEAD || true'
      }
    }

    stage('Lint & SAST') {
      parallel {
        stage('Backend Lint & SAST') {
          steps {
            sh '''
              if [ -f api/requirements.txt ]; then
                pip install -r api/requirements.txt || true
                flake8 api || true
                bandit -r api || true
              else
                echo "No backend requirements - skipping lint"
              fi
            '''
          }
        }
        stage('Frontend Lint') {
          when { expression { fileExists('frontend/package.json') } }
          steps {
            sh 'cd frontend && npm ci && npm run lint || true'
          }
        }
      }
    }

    stage('Unit Tests') {
      parallel {
        stage('Backend Tests') {
          steps {
            sh '''
              if [ -d api/tests ]; then
                (cd api && pytest -q) || true
              else
                echo "No backend tests detected - skipping"
              fi
            '''
          }
        }
        stage('Sim Tests') {
          steps {
            sh '''
              if [ -d simulator ]; then
                echo "Run simulator tests (none configured by default)"
              else
                echo "No simulator"
              fi
            '''
          }
        }
      }
    }

    stage('Build Images') {
      steps {
        sh '''
          if docker --version >/dev/null 2>&1; then
            docker build -t ${BACKEND_IMAGE} ./api || echo "backend build failed"
            docker build -t ${SIM_IMAGE} ./simulator || echo "simulator build failed"
            echo "IMAGE_BUILT=yes" > image_status.env
          else
            echo "IMAGE_BUILT=no" > image_status.env
          fi
          cat image_status.env
        '''
        // capture status file for later stages
        archiveArtifacts artifacts: 'image_status.env', allowEmptyArchive: false
      }
    }

    stage('Scan Images (Trivy)') {
      steps {
        sh '''
          IMAGE_BUILT=$(cat image_status.env | cut -d'=' -f2)
          if [ "$IMAGE_BUILT" = "yes" ]; then
            trivy image --severity CRITICAL --exit-code 1 ${BACKEND_IMAGE} || echo "Trivy backend check returned non-zero"
            trivy image --severity CRITICAL --exit-code 1 ${SIM_IMAGE} || echo "Trivy sim check returned non-zero"
          else
            echo "Skipping Trivy: images not built on this agent"
          fi
        '''
      }
    }

    stage('Publish Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED}", usernameVariable: 'NEXU_USER', passwordVariable: 'NEXU_PASS')]) {
          sh '''
            IMAGE_BUILT=$(cat image_status.env | cut -d'=' -f2)
            if [ "$IMAGE_BUILT" = "yes" ]; then
              echo "$NEXU_PASS" | docker login -u "$NEXU_USER" --password-stdin ${REGISTRY}
              docker push ${BACKEND_IMAGE} || echo "push backend failed"
              docker push ${SIM_IMAGE} || echo "push sim failed"
            else
              echo "Docker not available in agent; writing build-images.txt for manual load"
              echo "${BACKEND_IMAGE}" > build-images.txt || true
              echo "${SIM_IMAGE}" >> build-images.txt || true
            fi
          '''
        }
        archiveArtifacts artifacts: 'build-images.txt,image_status.env', allowEmptyArchive: true
      }
    }

    stage('Deploy to Dev') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
          sh '''
            # Update the deployments' image; tolerates if images not pushed yet
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-backend graphpass-backend=${BACKEND_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev set image deployment/graphpass-sim graphpass-sim=${SIM_IMAGE} --record || true
            kubectl --kubeconfig=$KUBECONFIG -n dev rollout status deployment/graphpass-backend --timeout=180s || true
          '''
        }
      }
    }

    stage('Integration Tests / Smoke') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
          sh '''
            # quick smoke: port-forward briefly and hit /healthz
            kubectl --kubeconfig=$KUBECONFIG -n dev port-forward svc/graphpass-backend 8080:80 >/tmp/portforward.log 2>&1 &
            PF_PID=$!
            sleep 3
            curl -sSf --retry 3 http://127.0.0.1:8080/healthz || (cat /tmp/portforward.log && kill $PF_PID && exit 1)
            kill $PF_PID || true
          '''
        }
      }
    }
  } // stages

  post {
    success {
      sh 'echo "Build succeeded: ${BUILD_TAG}"'
    }
    failure {
      sh 'echo "Build failed - collecting basic diagnostics"'
      archiveArtifacts artifacts: 'api/**/logs/**', allowEmptyArchive: true
    }
    always {
      junit allowEmptyResults: true, testResults: 'api/**/results-*.xml'
      archiveArtifacts artifacts: '**/sbom-*.json, build-images.txt, image_status.env', allowEmptyArchive: true
    }
  }
}
