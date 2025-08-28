pipeline {
  agent any
  options { disableConcurrentBuilds() }
  triggers { githubPush() }   // webhook is already working ✅

  environment {
    RECIPIENTS = 'atharvaupare5@gmail.com'   // TODO: change this
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        // make sure we have full history for diffs
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
        // Build a file list (A/M/D + path) into changes.txt
        bat '''
        @echo off
        setlocal ENABLEDELAYEDEXPANSION

        if not "%GIT_PREVIOUS_SUCCESSFUL_COMMIT%"=="" (
          set BASE=%GIT_PREVIOUS_SUCCESSFUL_COMMIT%
        ) else (
          for /f "usebackq delims=" %%a in (`git rev-parse HEAD~1`) do set BASE=%%a
        )

        git diff --name-status %BASE% HEAD > changes.txt

        if not exist changes.txt echo No changes detected compared to %BASE%>changes.txt

        endlocal
        '''
        script {
          env.CHANGED_FILES = readFile('changes.txt').trim()

          // Build a simple commits list using Jenkins changeSets
          def commits = []
          for (def cs in currentBuild.changeSets) {
            for (def e in cs.items) {
              commits << "- ${e.msg} (by ${e.author} @ ${e.commitId.take(7)})"
            }
          }
          if (commits.isEmpty()) {
            commits << "- Initial build or no recorded changeSets"
          }
          env.COMMITS_LIST = commits.unique().join('\n')
        }
      }
    }

    stage('Build/Test (optional)') {
      steps { echo 'Add build/test steps here if needed' }
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
${env.COMMITS_LIST ?: '- none -'}

Changed files (status\\tpath):
${env.CHANGED_FILES ?: '- none -'}
""".stripIndent()
      )
    }
  }
}
