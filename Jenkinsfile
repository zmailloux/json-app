def ENV_MAPPING = [ 'dev': [ 'env' : 'Development', 'app': 'json-app-dev'], 'uat': [ 'env' : 'QA', 'app': 'json-app-qa']]
def COLOR_MAP = ['SUCCESS': 'good', 'FAILURE': 'danger', 'UNSTABLE': 'danger', 'ABORTED': 'danger']


pipeline {
    agent { label 'master' }
    // Go to 'Manage Jenkins' -> 'Global Tool Configuration' and create the
    // following installations. If you give them different names you will need
    // to update the names in the 'tools' object below.
    tools {
        maven 'maven-3.5.4'
        jdk 'jdk-8'
        nodejs "NodeJS"
    }
    environment {
        BUILD_COLOR = ""
        //RELEASE_VERSION = "${ GIT_BRANCH.matches(".*\\d{1}(?:\\.\\d{1})+.*") ? GIT_BRANCH.split('/').first() : null }"
        //RELEASE_ENVIRONMENT = "${ GIT_BRANCH.matches(".*\\d{1}(?:\\.\\d{1})+.*") ? GIT_BRANCH.split('/')[1] : "dev" }"
        //BUILD_IDENTIFIER = "${BUILD_NUMBER}_${GIT_BRANCH}"
        API_NAME = "json-app"
    }

    stages {
        stage('Get Environment'){
          steps{
            script {
                if (BRANCH_NAME == 'dev/master') {
                    BUILD_IDENTIFIER = ""
                } else {
                    BUILD_IDENTIFIER = "-${GIT_BRANCH}"
                }
                BUILD_NAME = "0.1.${BUILD_NUMBER}${BUILD_IDENTIFIER}-SNAPSHOT"
            }
          }
        }

        stage('Build'){
            steps{
                // M2_SETTINGS are special maven settings we have that include
                // the enterprise libraries required to build the application.
                // Add the maven file to your Jenkins server and create a env
                // variable in 'Manage Jenkins' -> 'Configure System' -> 'Global properties'
                // and create 'M2_SETTINGS' with a path to your settings.xml file.
                sh "echo ${M2_SETTINGS}"
                sh "echo ${BUILD_NUMBER}"
                sh "mvn release:update-versions -DdevelopmentVersion=${BUILD_NAME} -s ${M2_SETTINGS}"
                // -DskipMunitTests is a temporary fix and should be removed
                sh "mvn -B clean verify -DskipMunitTests -s ${M2_SETTINGS}"
            }

            post {
                success {
                    publishHTML target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'target/munit-reports/coverage',
                        reportFiles: 'summary.html',
                        reportName: 'MUnit Report'
                    ]
                    publishHTML target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'JUnit Report'
                    ]
                }
            }
        }


        stage('Storing artifact'){
            stages{
                stage('Storing Artifact in Artifactory'){
                  steps{
                    script{
                      sh 'echo Sending JAR to artifactory'
                      sh 'echo ls target'
                      sh 'ls target'
                      // Artifactory pro
                      def server = Artifactory.server 'jfrog-pro'
                      def uploadSpec = """{
                        "files": [
                          {
                            "pattern": "target/*.zip",
                            "target": "${API_NAME}-dev/build/"
                          }
                       ]
                      }"""
                      def buildInfo = Artifactory.newBuildInfo()
                      buildInfo.env.collect()
                      buildInfo.name = "${API_NAME}${BUILD_IDENTIFIER}"
                      server.upload spec: uploadSpec, buildInfo: buildInfo
                      //buildInfo.env.collect()
                      //buildInfo.name = API_NAME
                      server.publishBuildInfo buildInfo
                      sh "echo ${BRANCH_NAME}"

                      if (BRANCH_NAME == 'dev/master') {
                        // Promotion logic
                        def promotionConfig = [
                            // Mandatory parameters
                            'buildName'          : buildInfo.name,
                            'buildNumber'        : buildInfo.number,
                            'targetRepo'         : 'json-app-prod',

                            // Optional parameters
                            'comment'            : 'this is the promotion comment',
                            'sourceRepo'         : 'json-app-dev',
                            'status'             : 'Released',
                            'includeDependencies': true,
                            'copy'               : true,
                            // 'failFast' is true by default.
                            // Set it to false, if you don't want the promotion to abort upon receiving the first error.
                            'failFast'           : true
                        ]

                        // Promote build configuration
                        Artifactory.addInteractivePromotion server: server, promotionConfig: promotionConfig, displayName: "Promote me please"
                      }

                    }
                  }
                }
              }
            }

        stage('Deployment'){
            stages{
                stage('Development - Deploy'){
                    when{
                        expression { GIT_BRANCH.matches(".*dev/master") && currentBuild.currentResult == 'SUCCESS' }
                    }
                    // Abstract out into another job
                    environment {
                        ANYPOINT_USERNAME = "zmailloux" // This needs to change to a ci account
                        // https://support.cloudbees.com/hc/en-us/articles/203802500-Injecting-Secrets-into-Jenkins-Build-Jobs
                        ANYPOINT_PASSWORD = credentials('anypoint-password')
                    }
                    steps{
                        // script{
                        //     ANYPOINT_ENV = ENV_MAPPING[RELEASE_ENVIRONMENT]['env']
                        // }
                        sh 'npm install anypoint-cli@latest'
                        // Original
                        //sh "./node_modules/anypoint-cli/src/app.js --environment=${ENV_MAPPING[RELEASE_ENVIRONMENT]['env']} runtime-mgr cloudhub-application modify ${ENV_MAPPING[RELEASE_ENVIRONMENT]['app']} target/${API_NAME}-1.0.${BUILD_NUMBER}-${RELEASE_ENVIRONMENT}-SNAPSHOT.zip"
                        // Test anypoint cli
                        sh "./node_modules/anypoint-cli/src/app.js --environment='Development' runtime-mgr cloudhub-application list"
                        // Modifies
                        sh "./node_modules/anypoint-cli/src/app.js --environment='Development' runtime-mgr cloudhub-application modify --property build.number:${API_NAME}-${BUILD_NAME} ${API_NAME}-dev target/${API_NAME}-${BUILD_NAME}.zip"

                    }
                  }
              }

              // post{
              //   success {
              //       script {
              //           if (fileExists("target/munit-reports/coverage")) {
              //               zip dir: "target/munit-reports/coverage", zipFile: "munit-report.zip"
              //           }
              //           if (fileExists("target/site/jacoco")) {
              //               zip dir: "target/site/jacoco", zipFile: "junit-report.zip"
              //           }
              //           stage "Create build output"
              //           archiveArtifacts artifacts: 'target/**/*.zip', fingerprint: true
              //       }
              //   }
              // }
        }
      }
      // cleanup {
      //     deleteDir()
      // }

    post {
      always {
        script {
          BUILD_COLOR = COLOR_MAP[currentBuild.currentResult]
          // DISPLAY_NAME = currentBuild.rawBuild.project.parent.displayName
        }
        // emailext(
        //   recipientProviders: [
        //     [$class: 'DevelopersRecipientProvider'],
        //     [$class: 'CulpritsRecipientProvider']
        //   ],
        //   subject: "${DISPLAY_NAME} | ${GIT_BRANCH}: ${currentBuild.currentResult}",
        //   body: "Check console output at ${BUILD_URL} to view the results.",
        //   attachLog: true
        // )
      //   slackSend channel: '#myrok-ci',
      //     color: BUILD_COLOR,
      //     message: "${DISPLAY_NAME} | ${BRANCH_NAME}: *${currentBuild.currentResult}* \nMore info at: ${BUILD_URL}"
      // }
      }
      success {
        script {
          if (fileExists("target/munit-reports/coverage")) {
            zip dir: "target/munit-reports/coverage", zipFile: "munit-report.zip"
          }
          if (fileExists("target/site/jacoco")) {
            zip dir: "target/site/jacoco", zipFile: "junit-report.zip"
          }
          if (fileExists("target/munit-reports/coverage") || fileExists("target/site/jacoco")) {
            emailext(
              recipientProviders: [
                [$class: 'DevelopersRecipientProvider'],
                [$class: 'CulpritsRecipientProvider']
              ],
              subject: "${JOB_BASE_NAME} | ${GIT_BRANCH}: Test Reports",
              body: "Test Reports attached",
              attachmentsPattern: "*-report.zip"
            )
          }
        }
      }

      cleanup {
        deleteDir()
      }
    }
}
