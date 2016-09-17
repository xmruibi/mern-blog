#!groovy

import groovy.json.JsonOutput

// Get all Causes for the current build
//def causes = currentBuild.rawBuild.getCauses()
//def specificCause = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause)

//echo "Cause: ${causes}"
//echo "SpecificCause: ${specificCause}"

stage 'DockerBuild'
slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Starting Docker Build"
node ('docker-cmd'){
    //env.PATH = "${tool 'Maven 3'}/bin:${env.PATH}"

    checkout scm

    sh "echo Working on BRANCH ${env.BRANCH_NAME} for ${env.BUILD_NUMBER}"

    sh './scripts/stamp_version.sh'

    dockerlogin()
    dockerrmi("xmruibi/mern-blog:${env.BRANCH_NAME}.${env.BUILD_NUMBER}")
    dockerbuild("xmruibi/mern-blog:${env.BRANCH_NAME}.${env.BUILD_NUMBER}")
}

stage 'Tests'
slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Tests"
node ('docker') {
    checkout scm

    dockerlogin()

    sh 'sudo npm config set color false; sudo npm install'

    sh './scripts/stamp_version.sh'


    env.MONGO_DB = "141_frontend_${env.BRANCH_NAME}_${env.BUILD_NUMBER}"
    env.ES_PREFIX = "${env.BRANCH_NAME}_${env.BUILD_NUMBER}_"

    sh './scripts/set_env_variables.sh; cat temp.env; . ./temp.env;  grunt test'

    step([$class: 'ArtifactArchiver', artifacts: 'reports/**,output/**', excludes: null])
    step([$class: 'JUnitResultArchiver', keepLongStdio: true, testResults: 'reports/*.xml'])
}

stage 'DockerHub'
slackSend color: 'green', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Pushing to Docker"
node('docker-cmd') {
    dockerlogin()
    retry ( 3 ) {
        dockerpush("xmruibi/mern-blog:${env.BRANCH_NAME}.${env.BUILD_NUMBER}")
    }

}

switch ( env.BRANCH_NAME ) {
    case "master":
        stage 'DockerLatest'
        slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Stopping DEV Services"
        node('docker-cmd') {
            dockerlogin()
            // Stop
            parallel (
                frontend1: { dockerstop('dev-service-frontend-001') },
            )

            slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Removing latest tag"
            // Erase
            dockerrmi('xmruibi/mern-blog:latest')

            // Tag
            dockertag("xmruibi/mern-blog:${env.BRANCH_NAME}.${env.BUILD_NUMBER}","xmruibi/mern-blog:latest")
            dockertag("xmruibi/mern-blog:${env.BRANCH_NAME}.${env.BUILD_NUMBER}","xmruibi/mern-blog:latest")

            // Push
            slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Pushing :latest"
            retry ( 3 ) {
                dockerpush('xmruibi/mern-blog:latest')
            }

            //docker -H tcp://10.1.10.210:5001 pull xmruibi/mern-blog:latest
        }

        stage 'Sleep'
        sleep 30

        slackSend color: 'blue', message: "ORG: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Building Downstream"

        break

    default:
        echo "Branch is not master.  Skipping tagging and push.  BRANCH: ${env.BRANCH_NAME}"
}


// Functions

// Docker functions
def dockerlogin() {
    sh "docker -H tcp://172.31.30.239:2375 login -e ${env.DOCKER_EMAIL} -u ${env.DOCKER_USER} -p ${env.DOCKER_PASSWD}"
}

def dockerbuild(label) {
    sh "docker -H tcp://172.31.30.239:2375 build -t ${label} ."
}
def dockerstop(vm) {
    sh "docker -H tcp://172.31.30.239:2375 stop ${vm} || echo stop ${vm} failed"
}

def dockerrmi(vm) {
    sh "docker -H tcp://172.31.30.239:2375 rmi -f ${vm} || echo RMI Failed"
}

def dockertag(label_old, label_new) {
    sh "docker -H tcp://172.31.30.239:2375 tag -f ${label_old} ${label_new}"
}

def dockerpush(image) {
    sh "docker -H tcp://172.31.30.239:2375 push ${image}"
}
