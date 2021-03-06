#!groovy

@Library('slackNotifications-shared-library@master') _

properties([
  parameters([
        string(name: 'apigeeUsername', defaultValue: 'bmutziu@gmail.com', description: 'ApiGee UserName'),
        password(name: 'apigeePassword', defaultValue: 'Apigee33916@', description: 'ApiGee Password'),
        string(name: 'base64encoded', defaultValue: 'Ym11dHppdUBnbWFpbC5jb206QXBpZ2VlMzM5MTZA', description: 'Username Password Combo Encoded')
        ])
])

final apigeeUsername = params.apigeeUsername
final apigeePassword = params.apigeePassword
final base64encoded = params.base64encoded

pipeline {
    agent any

    tools {
        maven 'm2'
        jdk 'openjdk'
        nodejs 'nodejs'
    }

    environment {
        //getting the current stable/deployed revision...this is used in undeploy.sh in case of failure...
        stable_revision = sh(script: 'curl -H "Authorization: Basic $base64encoded" "https://api.enterprise.apigee.com/v1/organizations/bmutziu-eval/apis/HR-API/deployments" | jq -r ".environment[0].revision[0].name"', returnStdout: true).trim()
    }

    stages {
        stage('Initial-Checks') {
            steps {
                sendNotifications 'STARTED'
                sh "npm -v"
                sh "mvn -v"
                echo "$apigeeUsername"
                echo "Stable Revision: ${env.stable_revision}"
        }}
        stage('Policy-Code Analysis') {
            steps {
                sh "npm install -g apigeelint"
                sh "apigeelint -s HR-API/apiproxy/ -f codeframe.js"
            }
        }
        stage('Unit-Test-With-Coverage') {
            steps {
                script {
                    try {
                        sh "npm install"
                        sh "npm test test/unit/*.js"
                        sh "npm run coverage test/unit/*.js"
                    } catch (e) {
                        throw e
                    } finally {
                        sh "cd coverage && cp cobertura-coverage.xml $WORKSPACE"
                        step([$class: 'CoberturaPublisher', coberturaReportFile: 'cobertura-coverage.xml'])
                    }
                }
            }
        }
/*        stage('Promotion') {
            steps {
                timeout(time: 2, unit: 'DAYS') {
                    input message: 'Do you want to Approve?', ok: 'Yes'
                }
            }
        }*/
        stage('Deploy to Production') {
            steps {
                 //deploy using maven plugin

                 //deploy only proxy and deploy both proxy and config based on edge.js update
                 //sh "sh && sh deploy.sh"
                sh "mvn -f HR-API/pom.xml install -Pprod -Dusername=${apigeeUsername} -Dpassword=${apigeePassword} -Dapigee.config.options=update"
            }
        }
        stage('Integration Tests') {
            steps {
                script {
                    try {
                        // using credentials.sh to get the client_id and secret of the app..
                        // thought of using them in cucumber oauth feature
                        // sh "sh && sh credentials.sh"
                        sh "cd $WORKSPACE/test/integration && npm install"
                        sh "cd $WORKSPACE/test/integration && npm test"
                    } catch (e) {
                        //if tests fail, I have used an shell script which has 3 APIs to undeploy, delete current revision & deploy previous stable revision
                        // sh "sh && sh undeploy.sh"
                        sh "echo Undeploying ..."
                        throw e
                    } finally {
                        // generate cucumber reports in both Test Pass/Fail scenario
                        sh "cd $WORKSPACE/test/integration && cp reports.json $WORKSPACE"
                        cucumber fileIncludePattern: 'reports.json'
                        //build job: 'apigee-cucumber-report'
                    }
                }
            }
        }
    }

    post {
        always {
            // cucumberSlackSend channel: 'testing', json: '$WORKSPACE/reports.json'
            sendNotifications currentBuild.result
        }
    }
}

/*

using shared library for slack reporting
    the lib groovy script must be placed in a vars folder in SCM
using build job to call a Freestyle project which sends the Cucumber reports to slack
    currently the cucumberSlackSend channel: 'apigee-cicd', json: '$WORKSPACE/reports.json'
        option doesnt send the reports to Slack
*/
