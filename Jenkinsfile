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
        // BRANCH_NAME = "${GIT_BRANCH.replaceAll('\\', '-')}"
        API_NAME = "json-app"
    }

    stages {
        stage('Get Environment'){
          steps{
            script {
                if (BRANCH_NAME == 'dev/master') {
                    BUILD_IDENTIFIER = ""
                } else {
                    // Replace /'s from the git branch
                    BUILD_IDENTIFIER = "-${GIT_BRANCH}"
                }
                BUILD_NAME = "1.0.${BUILD_NUMBER}${BUILD_IDENTIFIER}-SNAPSHOT"
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
                      server.publishBuildInfo buildInfo
                      sh "echo ${buildInfo}"

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
                steps{
                    sh "echo ABOUT TO RUN DEPLOY"
                    build job: 'zach-mule-deploy-dev', parameters: [[$class: 'StringParameterValue', name: 'api', value: "${API_NAME}"], [$class: 'StringParameterValue', name: 'zipFile', value: "${BUILD_NAME}.zip"]]
                }
              }
          }
        }
      }

    post {
      always {
        script {
          BUILD_COLOR = COLOR_MAP[currentBuild.currentResult]
        }
      }
      success {
        script {
          if (fileExists("target/munit-reports/coverage")) {
            zip dir: "target/munit-reports/coverage", zipFile: "munit-report.zip"
          }
          if (fileExists("target/site/jacoco")) {
            zip dir: "target/site/jacoco", zipFile: "junit-report.zip"
          }
        }
      }

      cleanup {
        deleteDir()
      }
    }
}
