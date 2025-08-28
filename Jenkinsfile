@NonCPS
String commitsAsText(changeSets) {
  def out = []
  changeSets.each { cs ->
    cs.items.each { e ->
      def authorName = e.getAuthor()?.getDisplayName()   // String only (no User objects)
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
  triggers { githubPush() }   // webhook → /github-webhook/

  environment {
    RECIPIENTS = 'atharvaupare5@gmail.com'   // <-- change this
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        // ensure full history for diffs
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
        // (1) name-status (A/M/D/R)  (2) numstat: +added \t -removed \t path
        bat '''
        @echo off
        setlocal ENABLEDELAYEDEXPANSION

        if not "%GIT_PREVIOUS_SUCCESSFUL_COMMIT%"=="" (
          set BASE=%GIT_PREVIOUS_SUCCESSFUL_COMMIT%
        ) else (
          for /f "usebackq delims=" %%a in (`git rev-parse HEAD~1`) do set BASE=%%a
        )

        git diff --name-status -M %BASE% HEAD > changes.txt
        git diff --numstat     -M %BASE% HEAD > numstat.tsv

        if not exist changes.txt  echo No changes detected compared to %BASE%>changes.txt
        if not exist numstat.tsv  echo No line changes detected compared to %BASE%>numstat.tsv

        endlocal
        '''
        script {
          // A/M/D list
          env.CHANGED_FILES = readFile('changes.txt').trim()

          // Commit summaries (safe: strings only)
          env.COMMITS_LIST = commitsAsText(currentBuild.changeSets)

          // Pretty table from numstat (tab-separated: added, removed, path)
          def raw   = readFile('numstat.tsv')
          def lines = raw.readLines()
          if (lines.isEmpty()) {
            env.NUMSTAT_TABLE = "No line changes detected"
          } else {
            def rows = []
            rows << String.format("%6s %8s  %s", "+added", "-removed", "file")
            rows << String.format("%6s %8s  %s", "------", "--------", "----")
            lines.each { ln ->
              def parts = ln.split("\\t")
              if (parts.size() >= 3) {
                rows << String.format("%6s %8s  %s", parts[0], parts[1], parts[2])
              } else {
                rows << ln
              }
            }
            env.NUMSTAT_TABLE = rows.join("\n")
          }
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
${env.COMMITS_LIST}

Changed files (status\\tpath):
${env.CHANGED_FILES}

Line changes (+added, -removed):
${env.NUMSTAT_TABLE}
""".stripIndent()
      )
    }
  }
}
