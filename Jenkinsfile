pipeline {
  agent any
  triggers { githubPush() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Test') {
      steps {
        echo 'Hello from Jenkinsfile!'
      }
    }
  }
}
