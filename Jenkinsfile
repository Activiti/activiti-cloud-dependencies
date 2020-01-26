pipeline {
    options {
        disableConcurrentBuilds()
        quietPeriod(30)
    }
    agent {
      label "jenkins-maven-java11"
    }
    environment {
      ORG               = 'activiti'
      APP_NAME          = 'activiti-cloud-dependencies'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      PREVIEW_NAMESPACE = "example-$BRANCH_NAME-$BUILD_NUMBER".toLowerCase().replaceAll("[\\-\\+\\.\\^:,]","");
      GLOBAL_GATEWAY_DOMAIN="35.242.205.159.nip.io"
      REALM = "activiti"
      GATEWAY_HOST = "gateway.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"
      SSO_HOST = "identity.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"
      GITHUB_CHARTS_REPO = "https://github.com/Activiti/activiti-cloud-helm-charts.git"

    }
    stages {
        stage('Configure Git Client') {
            steps {
                container('maven') {
                    sh "git config --global credential.helper store"
                    sh "jx step git credentials"
                }
            }
        }
        stage('Validate Versions') {
            steps {
                container('maven') {
                    sh "mvn validate"
                }
            }
        }
        stage('Build and Prepare Preview') {
            when {
                branch 'PR-*'
            }
            parallel {
                stage('PR Build Preview') {
                    steps {
                        container('maven') {
                            sh "mvn versions:set -DnewVersion=$PREVIEW_NAMESPACE"
                            sh "mvn install"
                        }
                    }
                }
                stage('PR Deploy Preview') {
                    steps {
                        container('maven') {
                            sh "make updatebot/push-version-dry"
                            sh "make prepare-helm-chart"
                            sh "make run-helm-chart"
                            sh "sleep 90"
                        }
                    }
                }
                stage("PR Prepare Acceptance Scenarios") {
                    steps {
                        container('maven') {
                            sh "make activiti-cloud-acceptance-scenarios"
                        }
                    }
                }
            }
        }
        stage("Run Acceptance Scenarios") {
            when {
                branch 'PR-*'
            }
            parallel {
                stage("Modeling Acceptance Tests") {
                    steps {
                        container('maven') {
                            sh "make modeling-acceptance-tests"
                        }
                    }
                    post {
                        failure {
                            slackSend(
                                channel: "#activiti-community-builds",
                                color: "danger",
                                message: "Modeling Acceptance Tests had failed: $BUILD_URL"
                            )
                        }
                    }
                }
                stage("Runtime Acceptance Scenarios") {
                    steps {
                        container('maven') {
                            sh "make runtime-acceptance-tests"
                        }
                    }
                    post {
                        failure {
                            slackSend(
                                channel: "#activiti-community-builds",
                                color: "danger",
                                message: "Runtime Acceptance Tests had failed: $BUILD_URL"
                            )
                        }
                    }
                }
            }
            post {
                success {
                    slackSend(
                        channel: "#activiti-community-builds",
                        message: "Runtime Acceptance Tests passed: $BUILD_URL"
                    )
                }
            }            
        }
      stage('Build Release') {
        when {
          branch 'develop'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout develop"

            // so we can retrieve the version in later steps
            sh "git fetch --tags"
            sh "echo \$(jx-release-version) > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
            sh "mvn clean verify"

            sh "make updatebot/push-version-dry"
            sh "make prepare-helm-chart"
            sh "make run-helm-chart"

            sh "make activiti-cloud-acceptance-scenarios"
            sh "sleep 90"

            sh "make modeling-acceptance-tests runtime-acceptance-tests"

            retry(5){
              sh "git add --all"
              sh "git commit -m \"Release \$(cat VERSION)\" --allow-empty"
              sh "git tag -fa v\$(cat VERSION) -m \"Release version \$(cat VERSION)\""
              sh "git push origin v\$(cat VERSION)"
            }
          }

          container('maven') {
            sh 'mvn clean deploy'

            sh 'export VERSION=`cat VERSION`'
            sh 'export UPDATEBOT_MERGE=false'

            sh "jx step git credentials"

            retry(2){
                sh "make updatebot/push-version"
            }
            script {
                def GIT_COMMIT_DETAILS = sh (
                    script: 'make git-rev-list',
                    returnStdout: true
                ).trim()
                println GIT_COMMIT_DETAILS

            slackSend(channel: "##activiti-community-builds", message: "New build propagated to AAE https://github.com/Alfresco/alfresco-process-parent/pulls ${GIT_COMMIT_DETAILS}" , sendAsText: true)
            }
          }
        }
        post {

            failure {
              slackSend(
                channel: "#activiti-community-builds",
                  color: "danger",
                  message: "Develop is failed $BUILD_URL"
              )
            }
        }
      }
      stage('helm chart release') {
              when {
                tag '*M*'
              }
              environment {
                //TAG_NAME = sh(returnStdout: true, script: 'git describe --always').trim()
                //HELM_ACTIVITI_VERSION = "$TAG_NAME"
                APP_ACTIVITI_VERSION = "$TAG_NAME"
                //APP_ACTIVITI_VERSION = "$BRANCH_NAME"
              }
              steps {
                container('maven') {
                    sh "echo $APP_ACTIVITI_VERSION >VERSION"
                    sh "git checkout $APP_ACTIVITI_VERSION"
                    sh "git config --global credential.helper store"
                    sh "jx step git credentials"
                    sh "git fetch --all --tags --prune"
                    sh "git checkout tags/$APP_ACTIVITI_VERSION -b $APP_ACTIVITI_VERSION"
                    //sh "make retag-docker-images"
                    //sh "make push-docker-images"
                    sh "make updatebot/push-version-dry"
                    sh "make replace-release-full-chart-names"
                    sh "make prepare-helm-chart"
                    sh "make run-helm-chart"

                    sh "make acc-tests"
                    sh "make github"
                    sh "make tag"
                }
              }
      }
    }
    post {
        always {
            delete_deployment()
            cleanWs()
        }
    }
}
def delete_deployment() {
  container('maven') {
   sh "make delete"
   sh "kubectl delete namespace $PREVIEW_NAMESPACE|echo 'try to remove namespace '$PREVIEW_NAMESPACE "
  }
}

