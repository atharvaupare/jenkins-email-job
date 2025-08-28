@NonCPS
String commitsAsText(changeSets) {
  def out = []
  changeSets.each { cs ->
    cs.items.each { e ->
      def authorName = e.getAuthor()?.getDisplayName()
      def msg        = e.getMsg()
      def shortSha   = e.getCommitId()?.take(7)
      out << "- ${msg} (by ${authorName} @ ${shortSha})"
    }
  }
  return out ? out.join('\n') : "- Initial build or no recorded changeSets"
}

pipeline {
  agent any
  options { disableConcurrentBuilds() }
  triggers { githubPush() }

  environment {
    RECIPIENTS = 'atharvaupare5@gmail.com'   // <-- change this
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        bat 'git fetch --all --prune'
        bat 'git rev-parse --is-shallow-repository > shallow.txt 2>NUL || echo false>shallow.txt'
        script {
          def shallow = readFile('shallow.txt').trim()
          if (shallow == 'true') { bat 'git fetch --unshallow' }
        }
      }
    }

    stage('Collect changes') {
      steps {
        bat '''
        @echo off
        setlocal ENABLEDELAYEDEXPANSION

        if not "%GIT_PREVIOUS_SUCCESSFUL_COMMIT%"=="" (
          set BASE=%GIT_PREVIOUS_SUCCESSFUL_COMMIT%
        ) else (
          for /f "usebackq delims=" %%a in (`git rev-parse HEAD~1`) do set BASE=%%a
        )

        git diff --name-status -M %BASE% HEAD > changes.txt

        if not exist changes.txt echo No changes detected compared to %BASE%>changes.txt

        endlocal
        '''
        script {
          env.CHANGED_FILES = readFile('changes.txt').trim()
          env.COMMITS_LIST  = commitsAsText(currentBuild.changeSets)
        }
      }
    }
  }

  post {
    always {
      emailext(
        to: env.RECIPIENTS,
        subject: "Jenkins #${BUILD_NUMBER} ${currentBuild.currentResult} — ${JOB_NAME}",
        body: """Build:  ${BUILD_URL}
Job:    ${JOB_NAME}
Branch: ${env.BRANCH_NAME ?: 'main'}
Result: ${currentBuild.currentResult}

Commits:
${env.COMMITS_LIST}

Changed files (status\\tpath):
${env.CHANGED_FILES}
""".stripIndent()
      )
    }
  }
}
