def ENV_MAPPING = [ 'dev': [ 'env' : 'Development', 'app': 'json-app-dev'], 'uat': [ 'env' : 'QA', 'app': 'json-app-qa']]


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
                sh "mvn release:update-versions -DdevelopmentVersion=1.0.${BUILD_NUMBER}${BUILD_IDENTIFIER} -s ${M2_SETTINGS}"
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

        stage('Deployment'){
            stages{
                stage('Development - Deploy'){
                    when{
                        not {
                            changeRequest()
                        }
                        expression { GIT_BRANCH.matches(".*/dev/master") && currentBuild.currentResult == 'SUCCESS' }
                    }
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
                        sh "./node_modules/anypoint-cli/src/app.js --environment='Development' runtime-mgr cloudhub-application modify --property build.number:${API_NAME}-1.0.${BUILD_NUMBER}${BUILD_IDENTIFIER} ${API_NAME}-dev target/${API_NAME}-1.0.${BUILD_NUMBER}${BUILD_IDENTIFIER}.zip"

                    }

                    post{
                      success {
                          script {
                              if (fileExists("target/munit-reports/coverage")) {
                                  zip dir: "target/munit-reports/coverage", zipFile: "munit-report.zip"
                              }
                              if (fileExists("target/site/jacoco")) {
                                  zip dir: "target/site/jacoco", zipFile: "junit-report.zip"
                              }
                              stage "Create build output"
                              archiveArtifacts artifacts: 'target/**/*.zip', fingerprint: true
                          }
                      }
                    }
                }
            }
        }


      }
}
