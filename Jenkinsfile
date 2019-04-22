
pipeline {
    agent { label 'master' }
    tools {
        maven 'maven-3.5.4'
        jdk 'jdk-8'
        nodejs "NodeJS"
    }
    environment {
        BUILD_COLOR = ""
        API_NAME = "json-app"
    }

    stages {
        stage('quick'){
            steps{
                sh 'ls'
            }
        }
        stage('Build'){
            steps{
                sh "mvn release:update-versions -DdevelopmentVersion=1.0.0"
                sh "echo ${BUILD_NUMBER}"
                //sh "mvn release:update-versions -DdevelopmentVersion=1.0.${BUILD_NUMBER}-${RELEASE_ENVIRONMENT}-SNAPSHOT -s ${M2_SETTINGS}"

                // -DskipMunitTests is a temporary fix and should be removed
                sh "mvn -B clean verify -DskipMunitTests"
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
      }
}
