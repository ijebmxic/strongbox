@Library('jenkins-shared-libraries') _

def REPO_NAME  = 'strongbox/strongbox'
def SERVER_ID  = 'carlspring-oss-snapshots'
def SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-snapshots/'

// Notification settings for "master" and "branch/pr"
def notifyMaster = [notifyAdmins: true, recipients: [culprits(), requestor()]]
def notifyBranch = [recipients: [brokenTestsSuspects(), requestor()]]

pipeline {
    agent {
        node {
            label 'alpine:jdk8-mvn3.6'
            customWorkspace workspace().getUniqueWorkspacePath()
        }
    }
    parameters {
        booleanParam(defaultValue: true, description: 'Send email notification?', name: 'NOTIFY_EMAIL')
        booleanParam(defaultValue: true, description: 'Trigger strongbox-os-build?', name: 'TRIGGER_OS_BUILD')
    }
    options {
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    stages {
        stage('Node')
        {
            steps {
                nodeInfo("mvn")
            }
        }
        stage('Building')
        {
            steps {
                withMavenPlus(timestamps: true, mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: '67aaee2b-ca74-4ae1-8eb9-c8f16eb5e534')
                {
                    // sh "mvn -U clean install -T2C -Dintegration.tests -Djunit.jupiter.execution.parallel.enabled=true -Djunit.jupiter.execution.parallel.config.strategy=fixed -Djunit.jupiter.execution.parallel.config.fixed.parallelism=2 -Dprepare.revision -Dmaven.test.failure.ignore=true -Pcoverage"
                    sh "mvn -U clean install -T2C -Dintegration.tests -Dprepare.revision -Dmaven.test.failure.ignore=true -Pbuild-rpm,coverage"
                }
            }
        }
        stage('Code Analysis') {
            steps {
                withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: '67aaee2b-ca74-4ae1-8eb9-c8f16eb5e534', publisherStrategy: 'EXPLICIT')
                {
                    withCredentials([
                        string(credentialsId: '5aa5789f-dd6a-48c2-a76c-10d8b16a4e53', variable: 'CODACY_API_TOKEN'),
                        string(credentialsId: 'b3d644ac-5a8c-4a07-bf6e-6953a46ac33f', variable: 'CODACY_PROJECT_TOKEN_STRONGBOX')
                    ]) {
                        sh "mvn com.gavinmogan:codacy-maven-plugin:coverage -Pcodacy"
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                expression { BRANCH_NAME == 'master' && (currentBuild.result == null || currentBuild.result == 'SUCCESS') }
            }
            steps {
                script {
                    withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833', publisherStrategy: 'EXPLICIT')
                    {
                        sh "mvn deploy" +
                           " -DskipTests" +
                           " -DaltDeploymentRepository=${SERVER_ID}::default::${SERVER_URL}"
                    }
                }
            }
        }
        stage('Deploying to GitHub') {
            when {
                expression { BRANCH_NAME == 'master' && (currentBuild.result == null || currentBuild.result == 'SUCCESS') }
            }
            steps {
                withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833', publisherStrategy: 'EXPLICIT')
                {
                    sh "mvn package -Pdeploy-release-artifact-to-github -Dmaven.test.skip=true"
                }
            }
        }
    }
    post {
        success {
            script {
                if(BRANCH_NAME == 'master' && params.TRIGGER_OS_BUILD) {
                    build job: "strongbox/strongbox-os-builds", wait: false, parameters: [[$class: 'StringParameterValue', name: 'REVISION', value: '*/master']]
                }
            }
        }
        failure {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyFailed((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        unstable {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyUnstable((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        fixed {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyFixed((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        always {
            // (fallback) record test results even if withMaven should have done that already.
            junit '**/target/*-reports/*.xml'
        }
        cleanup {
            script {
                workspace().clean()
            }
        }
    }
}
