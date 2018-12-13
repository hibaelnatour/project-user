#!/usr/bin/env groovy
// https://github.com/freebsd/freebsd-ci/blob/master/scripts/build/build-test.groovy
// http://stackoverflow.com/a/40294220

// https://JENKINS_HOST/scriptApproval/ - for script approval

import java.util.Date

def isMaster = env.BRANCH_NAME == 'master'
def isDevelop = env.BRANCH_NAME == 'develop'
def start = new Date()
def err = null

String slackChannel = isMaster ? 'CI-PROD' : 'CI-STAGING'  // slack channels for notifications
String repo = 'REPOSITORY_NAME'  // TODO take from env
String sshCredentialsId = 'JENKINS_SSH_CREDENTIALS_ID'
String jobInfoShort = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME}"
String jobInfo = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME} \n${env.BUILD_URL}"
String workspace
String buildStatus
String timeSpent
String changelog
String commitHash
String commitText
String deployCmd

currentBuild.result = "SUCCESS"

try {

node {

    workspace = pwd()
    env.PATH = "${workspace}/env/bin:/usr/bin:${env.PATH}"  // for python projects
    // env.PATH = "${workspace}/node_modules/.bin:/usr/bin:${env.PATH}"  // for nodejs projects

    // clean up previous build
    stage (name : 'Cleanup') {
        sh "test -d ${workspace}/env && rm -rf ${workspace}/env || echo 'no env, skipping cleanup'"  // for python projects
        // sh "test -d ${workspace}/dist && rm -rf ${workspace}/dist || echo 'no dist dir, skipping cleanup'"  // for nodejs projects
    }

    // Mark the code checkout 'stage'
    stage (name : 'Checkout') {
        // Get the code from the repository
        checkout scm
    }

    changelog = changeLogs()
    if (changelog) {
        changelog = "Changes in *${repo}* repository *${env.BRANCH_NAME}* branch detected\n" + changelog
        slackSend channel: slackChannel, message: "${changelog}"
    } else {
        commitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(7)
        commitText = sh(returnStdout: true, script: 'git show -s --format=format:"*%s*  _by %an_" HEAD').trim()
        changelog = "No changes in *${repo}* repository *${env.BRANCH_NAME}* branch detected\n"
        changelog = changelog + "Building for commit: \n`${commitHash}` ${commitText}"
        slackSend channel: slackChannel, message: "${changelog}"
    }
    slackSend channel: slackChannel, color: "#439FE0", message: "Build started: ${jobInfoShort}"

    // Mark the code build 'stage'
    stage (name : 'Build') {
        // for python projects
        sh 'virtualenv env'
        sh "${workspace}/env/bin/pip install -r requirements/prod.txt"
      
        // for nodejs projects
        // sh 'npm install'
        // sh 'npm install bower'
        // sh 'bower install'
        // sh 'npm run build'
    }

    // Start the tests
    // withEnv(["DB_HOST=localhost",
    //          "DB_USER=",
    //          "DB_PASS=",
    //          "DB_NAME=${repo}db_${env.BRANCH_NAME}",
    //          "DB_PORT=5432"]) {
    //     stage (name : 'Test') {
    //         sh "source ${workspace}/env/bin/activate && ./test --noinput"  // for python projects
    //         // sh 'npm run test'  // for nodejs projects
    //     }
    // }

    if (isDevelop || isMaster) {
        // for python projects
        deployCmd = isMaster ? 'fab deploy_prod' : 'fab deploy_staging'
        sshagent([sshCredentialsId]) {
            stage (name : 'Deploy') {
                sh "source ${workspace}/env/bin/activate && ${deployCmd}"
            }
        }
        // TODO for nodejs projects
    }

}

} catch (caughtError) {

    err = caughtError
    currentBuild.result = "FAILURE"

} finally {

    timeSpent = "\nTime spent: ${timeDiff(start)}"

    if (err) {
        slackSend channel: slackChannel, color: 'danger', message: "Build failed: ${jobInfo} ${timeSpent}"
        throw err
    } else {
        if (currentBuild.previousBuild == null) {
            buildStatus = 'First time build'
        } else if (currentBuild.previousBuild.result == 'SUCCESS') {
            buildStatus = 'Build complete'
        } else {
            buildStatus = 'Back to normal'
        }
        slackSend channel: slackChannel, color: 'good', message: "${buildStatus}: ${jobInfo} ${timeSpent}"

        if (isDevelop || isMaster) {
            slackSend channel: slackChannel, color: 'good', message: "*${env.BRANCH_NAME}* branch deployed"
        }
    }
}

def timeDiff(st) {
    def delta = (new Date()).getTime() - st.getTime()
    def seconds = delta.intdiv(1000) % 60
    def minutes = delta.intdiv(60 * 1000) % 60

    return "${minutes} min ${seconds} sec"
}

@NonCPS
def prevBuildLastCommitId() {
    def prev = currentBuild.previousBuild
    def items = null
    def result = null
    if (prev != null && prev.changeSets != null && prev.changeSets.size() && prev.changeSets[prev.changeSets.size() - 1].items.length) {
        items = prev.changeSets[prev.changeSets.size() - 1].items
        result = items[items.length - 1].commitId
    }
    return result
}

@NonCPS
def commitInfo(commit) {
    return commit != null ? "`${commit.commitId.take(7)}`  *${commit.msg}*  _by ${commit.author}_ \n" : ""
}

@NonCPS
def changeLogs() {
    String msg = ""
    String repoUrl = 'BITBUCKET_REPO_URL'
    def lastId = null
    def prevId = prevBuildLastCommitId()
    def changeLogSets = currentBuild.changeSets

    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            lastId = entry.commitId
            msg = msg + "${commitInfo(entry)}"
        }
    }
    if (prevId != null && lastId != null) {
        msg = msg + "\n${repoUrl}/branches/compare/${lastId}..${prevId}#diff\n"
    }
    return msg
}