pipeline {
  agent any

  options {
    copyArtifactPermission('*')
  }

  stages {
    stage('Check Out') {
      steps {
        git(url: 'https://github.com/JugalBili/maven-samples-A6.git', branch: 'master')
      }
    }

    stage('Verify') {
      steps {
        catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
          sh """
            #!/bin/bash
            set +e   # Disable 'exit immediately on error'

            CURRENT_COMMIT=\$(git rev-parse HEAD)

            mvn clean verify
            RESULT=\$?

            if [ "\$RESULT" -eq 0 ]; then
              # Build succeeded, update latest success hash
              echo "\$CURRENT_COMMIT" > latest_success_hash.txt
              echo "Build succeeded. Updated latest_success_hash to \$CURRENT_COMMIT"
            else
              echo "Build failed. Not updating latest_success_hash."
            fi

            exit \$RESULT
          """
        }

        script {
          if (fileExists('latest_success_hash.txt')) {
            echo "New latest_success_hash.txt file found, uploading new artifact"
            archiveArtifacts artifacts: 'latest_success_hash.txt', fingerprint: true
          } else {
            echo "Build failed, reuploading previous artifact"
            try {
              copyArtifacts filter: "latest_success_hash.txt", fingerprintArtifacts: true, projectName: env.JOB_NAME, selector: lastCompleted()
            } catch (Exception e) {
              echo "No previous build found. Skipping artifact retrieval: ${e}"
            }

            archiveArtifacts artifacts: 'latest_success_hash.txt', fingerprint: true
          }
        }
      }
    }

    stage('Git Bisect') {
      steps {
        script {
          // Try to retrieve the latest_success_hash.txt from the last successful build
          try {
            copyArtifacts filter: "latest_success_hash.txt", fingerprintArtifacts: true, projectName: env.JOB_NAME, selector: lastCompleted()
          } catch (Exception e) {
            echo "No previous build found. Skipping artifact retrieval: ${e}"
          }

          sh """
            # Get the current commit hash
            CURRENT_COMMIT=\$(git rev-parse HEAD)

            # Load the latest success hash if exists
            if [ -f latest_success_hash.txt ]; then
              LATEST_SUCCESS_HASH=\$(cat latest_success_hash.txt)
            else
              LATEST_SUCCESS_HASH=""
            fi
            echo "Latest Successful Commit: \$LATEST_SUCCESS_HASH"

            # Run Git Bisect if hashes don't match and latest_success_hash exists
            if [ -n "\$LATEST_SUCCESS_HASH" ] && [ "\$LATEST_SUCCESS_HASH" != "\$CURRENT_COMMIT" ]; then
              echo "Running Git Bisect to find the first bad commit..."
              git bisect start "\$CURRENT_COMMIT" "\$LATEST_SUCCESS_HASH"
              git bisect run mvn clean verify
              git bisect reset
            else
              echo "No need to run Git Bisect."
            fi
          """
        }
      }
    }
  }

  tools {
    maven 'A6_MVN'
    jdk 'A6_JDK'
  }
}