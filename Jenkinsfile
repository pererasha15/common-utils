pipeline {
    agent any
    environment {
        GITHUB_PAT = credentials('git-pat-26-2')
    }
    stages {
        stage('Build') {
            steps {
                echo 'Building..'
                sh '''
                  ls -la
                  printenv
                  ./mvnw -version
                  ./mvnw clean test package
                '''
            }
        }
        stage('Deploy') {
            steps {
                sh 'printenv'
                script {
                  def version = sh(returnStdout: true, script: './mvnw help:evaluate -Dexpression=project.version -q -DforceStdout').trim()
                  echo "version=$version"

                  def commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
                  echo "git last commit message=$commitMessage"

                  def skipRelease = (commitMessage ==~ /^(JIRA:MAINT-00000)\s.+$/)

                  if (skipRelease) {
                    echo "*** Skipping release ***"
                  } else {
                    if ('main'.equals(env.BRANCH_NAME)) {
                      echo "*** Deploying to release repo ***"
                      withCredentials([gitUsernamePassword(credentialsId: 'git-pat-26-2', gitToolName: 'git-tool')]) {
                        try {
                          sh './mvnw release:prepare -B -Dresume=false -DscmCommentPrefix="JIRA:MAINT-00000 "'
                          sh './mvnw release:perform'
                        } catch (err) {
                          sh './mvnw release:rollback'
                          sh 'git push --delete origin $(git describe --abbrev=0 --tags)'
                          throw err
                        }
                      }
                    } else {
                      echo "*** Deploying to snapshot repo ***"
                      def safeBranch = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9]', '-')
                      def versionName = version.replace("-SNAPSHOT", "") + "-" + safeBranch + "-SNAPSHOT"

                      sh '''
                        ./mvnw versions:set -DnewVersion=${versionName}"
                        ./mvnw deploy
                      '''
                    }
                  }
                }
            }
        }
    }
}