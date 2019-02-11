pipeline {

  agent {
    label 'jenkins-maven-java11'
  }

  environment {
    // application
    ORG            = 'interdiscount'
    APP_NAME        = 'inventory-service'
    // jenkins
    CONTAINER_ID      = 'maven-java11'
    // helm
    CHARTMUSEUM_CREDS    = credentials('jenkins-x-chartmuseum')
    CHART_FOLDER      = "./charts"
    CHART_APP_FOLDER    = "$CHART_FOLDER/$APP_NAME"
    // git
    MASTER_BRANCH_NAME    = 'master'
    FEATURE_BRANCH_NAME    = /feature\/.*/
    PR_BRANCH_NAME_REGEX  = /PR-.*/
    PR_BRANCH_NAME      = 'PR-*'
    VERSION_ALIGNMENT_MSG  = 'Version alignment commit'
  }

  stages {

    /*   ***   For the debug stages have a look at the bottom */

    /*   ***   PRELIMINARY stages   ***   */

    stage('Git commit message check') {
      steps {
        script {
          // Retrieve last git commit message
          LAST_COMMIT_MSG = sh(
            script: 'git log --oneline --format=%B -n 1 HEAD',
            returnStdout: true
          ).trim()

          // Verify if just a version alignment commit
          VERSION_ALIGN_COMMIT = BRANCH_NAME.equals(MASTER_BRANCH_NAME) && LAST_COMMIT_MSG.contains(VERSION_ALIGNMENT_MSG)

          // Verify if a feature
          IS_FEATURE_BRANCH = BRANCH_NAME ==~ /$FEATURE_BRANCH_NAME/

          // Verify if a PR
          IS_PR = BRANCH_NAME ==~ /$PR_BRANCH_NAME_REGEX/

          // echo "[DEBUG] branch name: ${BRANCH_NAME}"
          // echo "[DEBUG] commit message: ${LAST_COMMIT_MSG}"
          // echo "[DEBUG] version alignment commit: ${VERSION_ALIGN_COMMIT}"
          // echo "[DEBUG] is feature branch: ${IS_FEATURE_BRANCH}"
          // echo "[DEBUG] is PR: ${IS_PR}"
        }
      }
    }


    /*   ***   PR PREVIEW stages   ***   */

    stage('PR previews') {
      when {
        branch PR_BRANCH_NAME
      }
      environment {
        CHART_PREVIEW_FOLDER = "$CHART_FOLDER/preview"
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        container(CONTAINER_ID) {
          // Prepare artifact
          sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
          sh 'mvn install -DskipTests'
          // sh 'echo [DEBUG] prepare artifact: DONE'

          // Prepare Docker image
          sh 'export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml'
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          // sh 'echo [DEBUG] prepare docker image: DONE'

          dir ("$CHART_PREVIEW_FOLDER") {
            // Set preview version in JenkinsX Chart.yaml, values.yaml and requirements.yaml
            script {
              // Retrieve OS version
              OS = sh(
                script: 'uname',
                returnStdout: true
              ).trim()

              // echo "[DEBUG] os: ${OS}"

              if ( OS.equals('Darwin') ) {
                sh(
                  script: "sed -i '' -e 's/version:.*/version: ${PREVIEW_VERSION}/' Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i '' -e 's/version:.*/version: ${PREVIEW_VERSION}/' ../*/Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i '' -e 's/tag: .*/tag: ${PREVIEW_VERSION}/' values.yaml",
                  returnStdout: true
                ).trim()

              } else if ( OS.equals('Linux') ) {
                sh(
                  script: "sed -i -e 's/version:.*/version: ${PREVIEW_VERSION}/' Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i -e 's/version:.*/version: ${PREVIEW_VERSION}/' ../*/Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i -e 's|repository: .*|repository: ${DOCKER_REGISTRY}/${ORG}/${APP_NAME}|' values.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i -e 's/tag: .*/tag: ${PREVIEW_VERSION}/' values.yaml",
                  returnStdout: true
                ).trim()

              } else {
                echo "Platfrom ${OS} not supported to release from, exiting pipeline..."
                exit -1
              }

              sh(
                script: "echo '  version: ${PREVIEW_VERSION}' >> requirements.yaml",
                returnStdout: true
              ).trim()
            }
            // sh 'echo [DEBUG] set preview version in files: DONE'

            // Prepare helm chart
            sh 'jx step helm build'
            // sh 'echo [DEBUG] prepare helm chart: DONE'

            // Deploy preview
            sh "jx preview --app $APP_NAME --dir ../.."
            // sh 'echo [DEBUG] deploy: DONE'
          } // dir
        } // container
      } // steps
    } // stage


    // /*   ***   VERSION ALIGNMENT stages   ***   */

    stage('Version alignment') {
      when {
        expression {
          VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        container(CONTAINER_ID) {
          sh 'echo This is just a commit to align maven, helm and jx versions'
          sh 'echo Nothing will be verified, compiled, tested or deployed'
          sh 'echo The pipeline will end after this stage'
        }
      }
    }


    /*   ***   MASTER & FEATURE stages   ***   */

    stage('Checkout') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR
        }
      }
      steps {
        container(CONTAINER_ID) {
          sh "git checkout $BRANCH_NAME"
          sh 'git config --global credential.helper store'
          sh "jx version --batch-mode"
          sh 'jx step git credentials'
        }
      }
    }

    stage('Compile') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR
        }
      }
      steps {
        container(CONTAINER_ID) {
          sh 'mvn clean compile'
        }
      }
    }

    stage('Unit tests') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR
        }
      }
      steps {
        container(CONTAINER_ID) {
          sh 'mvn -Dtest=*UnitTest test'
        }
      }
    }

    stage('Smoke tests') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR
        }
      }
      steps {
        container(CONTAINER_ID) {
          sh 'mvn -Dtest=*StartupTest test'
        }
      }
    }

    // stage('Integration tests') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'mvn -Dtest=*IntegrationTest test'
    //     }
    //   }
    // }

    // stage('Functional tests') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'mvn -Dtest=*FunctionalTest -DfailIfNoTests=false test'
    //     }
    //   }
    // }

    // stage('Contract tests') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'mvn -Dtest=*ContractTest -DfailIfNoTests=false test'
    //     }
    //   }
    // }

    // stage('Code coverage analysis') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'mvn jacoco:report jacoco:check'
    //     }
    //   }
    // }

    // stage('Mutation tests') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'mvn pitest:mutationCoverage'
    //     }
    //   }
    // }
    
    // TODO
    // stage('Libraries vulnerabilities check') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'echo This is just a DRAFT step, no libraries vurnerabilities check for now...'
    //     }
    //   }
    // }

    // TODO
    // stage('Static code analysis') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'echo This is just a DRAFT step, no static code analysis for now...'
    //       // ? pmd
    //       // ? findbugs
    //       // ? checkstyle
    //       // sh 'mvn sonar:sonar'
    //       // TODO quality gate check
    //     }
    //   }
    // }

    /*   ***   MASTER ONLY stages   ***   */

    // TODO
    // stage('Performance tests') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'echo This is just a DRAFT step, no performance tests for now...'
    //     }
    //   }
    // }

    stage('Identify next version') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        container(CONTAINER_ID) {
          // Identify version
          sh 'jx step next-version --use-git-tag-only=true'
          sh 'echo Next version to be released: \$(cat VERSION)'

          // Set new version in maven pom
          sh 'mvn versions:set -DnewVersion=\$(cat VERSION)'

          dir ("$CHART_APP_FOLDER") {
            // Set new version in helm chart
            // /!\   WARNINGS   /!\
            // 1. changing the value in Chart.yaml, but lost due to 'jx step changelog' command
            // 2. it is still needed, in order to let helm deploy the right version to the chartmuseum
            // 3. this is triggering another time the pipeline leading to an infinite loop?
            sh 'jx step helm version --version \$(cat ../../VERSION)'

            // Set new version in JenkinsX Chart.yaml and values.yaml
            script {
              // Retrieve OS version
              OS = sh(
                script: 'uname',
                returnStdout: true
              ).trim()
              // Set release version in an environment variable
              RELEASE_VERSION = sh(
                script: 'cat ../../VERSION',
                returnStdout: true
              ).trim()

              // echo "[DEBUG] os: ${OS}"
              // echo "[DEBUG] version: ${RELEASE_VERSION}"

              if ( OS.equals('Darwin') ) {
                sh(
                  script: "sed -i '' -e 's/version:.*/version: ${RELEASE_VERSION}/' Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i '' -e 's/tag: .*/tag: ${RELEASE_VERSION}/' values.yaml",
                  returnStdout: true
                ).trim()

              } else if ( OS.equals('Linux') ) {
                sh(
                  script: "sed -i -e 's/version:.*/version: ${RELEASE_VERSION}/' Chart.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i -e 's|repository: .*|repository: ${DOCKER_REGISTRY}/${ORG}/${APP_NAME}|' values.yaml",
                  returnStdout: true
                ).trim()
                sh(
                  script: "sed -i -e 's/tag: .*/tag: ${RELEASE_VERSION}/' values.yaml",
                  returnStdout: true
                ).trim()

              } else {
                echo "Platfrom ${OS} not supported to release from, exiting pipeline..."
                exit -1
              }
            } // script

            // sh '[DEBUG] cat Chart.yaml'
            // sh '[DEBUG] cat values.yaml'
          } // dir
        } // container
      } // steps
    } // stage

    stage('Create repository tag') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        container(CONTAINER_ID) {
          // Creates a git tag
          // PLEASE NOTE: this is not updating ./charts/values.yaml file with the right docker image repository (default 'draft') and tag (defaul 'dev')
          sh 'jx step tag --version \$(cat VERSION)'
         }
      }
    }

    stage('Create repository release') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        dir ("$CHART_APP_FOLDER") { 
          container(CONTAINER_ID) {          

            // Creates a changelog for a git tag          
            sh "jx step changelog --version v\$(cat ../../VERSION)"
           }
        }        
      }
    }

    stage('Deploy artifact') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        container(CONTAINER_ID) {
          // PLEASE NOTE: goal 'deploy:deploy' does not act same as phase 'deploy'
          // see https://stackoverflow.com/a/38148135
          sh 'mvn -DskipTests -Djacoco.skip=true deploy'
        }
      }
    }

    stage('Release container image') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        container(CONTAINER_ID) {
          // Create docker image using skaffold
          sh 'export VERSION=\$(cat VERSION) && skaffold build -f skaffold.yaml'

          // Push image to docker registry
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        }
      }
    }

    // TODO
    // stage('Container vurnerabilities check') {
    //   when {
    //     expression {
    //       !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
    //     }
    //   }
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'echo This is just a DRAFT step, no container vurnerabilities check for now...'
    //     }
    //   }
    // }

    stage('Release chart') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        dir ("$CHART_APP_FOLDER") {
          container(CONTAINER_ID) {
            // Release the helm chart
            sh 'jx step helm release'
          }
        }
      }
    }

    stage('Promote to environments') {
      when {
        expression {
          !VERSION_ALIGN_COMMIT && !IS_PR && !IS_FEATURE_BRANCH
        }
      }
      steps {
        dir ("$CHART_APP_FOLDER") {
          container(CONTAINER_ID) {
            // Promoting through all auto-promotion Environments
            sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
          }
        }
      }
    }

    /*   ***   DEBUG stages   ***   */

    // stage('Debug stage - Current build groovy variable') {
    //   steps {
    //     script {
    //       echo "Current build: ${currentBuild}"
    //       echo "Change sets: ${currentBuild.changeSets}"
    //     }
    //   }
    // }

    // stage('Debug stage - Custom env vars') {
    //   steps {
    //     script {
    //       echo "org: ${ORG}"
    //       echo "app name: ${APP_NAME}"
    //       echo "chart museum creds: ${CHARTMUSEUM_CREDS}"
    //       echo "chart folder: ${CHART_FOLDER}"
    //       echo "chart app folder: ${CHART_APP_FOLDER}"
    //       echo "master branch name: ${MASTER_BRANCH_NAME}"
    //       echo "features branch name: ${FEATURE_BRANCH_NAME}"
    //       echo "pull requests branch name: ${PR_BRANCH_NAME}"
    //     }
    //   }
    // }

    // stage('Debug stage - Global env vars') {
    //   steps {
    //     container(CONTAINER_ID) {
    //       sh 'echo Environment variables:'
    //       sh 'env'
    //     }
    //   }
    // }

  } // stages

  post {
    always {
      cleanWs()
    }
    failure {
        input '''
Pipeline failed.
We will keep the build pod around to help you diagnose any failures.

Select 'Proceed' or 'Abort' to terminate the build pod.'''
    }
  }

} // pipeline


/*  ***   LINKS   ***
  https://jenkins.io/doc/book/pipeline/jenkinsfile/
  https://jenkins.io/doc/book/pipeline/syntax/
  https://stackoverflow.com/questions/44870978/how-to-run-multiple-stages-on-the-same-node-with-declarative-jenkins-pipeline
  https://stackoverflow.com/questions/37690920/jenkins-pipeline-conditional-step-stage
  http://mrhaki.blogspot.com/2009/09/groovy-goodness-matchers-for-regular.html
*/

/*  ***   PLUGINS   ***
  Parallel Test Executor
    https://wiki.jenkins.io/display/JENKINS/Parallel+Test+Executor+Plugin
  Matrix-based security
    https://wiki.jenkins.io/display/JENKINS/Matrix-based+security
  Embeddable Build Status
    https://wiki.jenkins.io/display/JENKINS/Embeddable+Build+Status+Plugin
*/
