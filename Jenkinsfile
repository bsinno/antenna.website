// See documentation on: https://wiki.eclipse.org/Jenkins#Build_my_project.27s_website_with_Jenkins.3F

pipeline {

    agent {
        kubernetes {
            label 'hugo-agent'
            yaml """
apiVersion: v1
metadata:
  labels:
    run: hugo
  name: hugo-pod
spec:
  containers:
    - name: jnlp
      volumeMounts:
      - mountPath: /home/jenkins/.ssh
        name: volume-known-hosts
      env:
      - name: "HOME"
        value: "/home/jenkins/agent"
    - name: hugo
      image: eclipsecbi/hugo:0.42.1
      command:
      - cat
      tty: true
  volumes:
  - configMap:
      name: known-hosts
    name: volume-known-hosts
"""
        }
    }

    environment {
        PROJECT_NAME = "antenna" // must be all lowercase.
        PROJECT_BOT_NAME = "Antenna Bot" // Capitalize the name
    }

    triggers { pollSCM('H/10 * * * *')}

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        checkoutToSubdirectory('hugo')
    }

    stages {
        stage('Checkout www repo') {
            steps {
                dir('www') {
                    sshagent(['git.eclipse.org-bot-ssh']) {
                        sh '''
                            git clone ssh://genie.${PROJECT_NAME}@git.eclipse.org:29418/www.eclipse.org/${PROJECT_NAME}.git .
                            git checkout ${BRANCH_NAME}
                        '''
                    }
                }
            }
        }
        stage('Build website (master) with Hugo') {
            when {
                branch 'master'
            }
            steps {
                container('hugo') {
                    dir('hugo') {
                        sh 'hugo -b https://www.eclipse.org/${PROJECT_NAME}/'
                    }
                }
            }
        }
        stage('Build website (staging) with Hugo') {
            when {
                branch 'staging'
            }
            steps {
                container('hugo') {
                    dir('hugo') {
                        sh 'hugo -b https://staging.eclipse.org/${PROJECT_NAME}/'
                    }
                }
            }
        }
        stage('Push to $env.BRANCH_NAME branch') {
            when {
                anyOf {
                    branch "master"
                    branch "staging"
                }
            }
            steps {
                sh 'rm -rf www/* && cp -Rvf hugo/public/* www/'
                dir('www') {
                    sshagent(['git.eclipse.org-bot-ssh']) {
                        sh '''
                        git add -A
                        if ! git diff --cached --exit-code; then
                          echo "Changes have been detected, publishing to repo 'www.eclipse.org/${PROJECT_NAME}'"
                          git config --global user.email "${PROJECT_NAME}-bot@eclipse.org"
                          git config --global user.name "${PROJECT_BOT_NAME}"
                          git commit -m "Website build ${JOB_NAME}-${BUILD_NUMBER}"
                          git log --graph --abbrev-commit --date=relative -n 5
                          git push origin HEAD:${BRANCH_NAME}
                        else
                          echo "No change have been detected since last build, nothing to publish"
                        fi
                        '''
                    }
                }
            }
        }
    }
}
