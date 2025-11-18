pipeline {
  agent any

  stages {
    stage('Debug Agent') {
      steps {
        sh 'hostname && printf "\\n=== WHOAMI ===\\n" && whoami || id || echo "no whoami"'
      }
    }
  }
}
