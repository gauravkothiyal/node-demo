#!/usr/bin/env groovy
import java.text.SimpleDateFormat
import groovy.json.*

class BuildImage {
    def environment
}

properties(
        [
        disableConcurrentBuilds(),
        buildDiscarder(
            logRotator(artifactDaysToKeepStr: '1',
            artifactNumToKeepStr: '10',
            daysToKeepStr: '1',
            numToKeepStr: '10'
            )
                ),
                [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
        ]             
)

// NEXUS
def RepositoryName = 'node-demo'
def HarborUrl = "10.39.1.159:2880"
def HarborRegistry = "node-demo"
def credentials = com.cloudbees.plugins.credentials.CredentialsProvider.lookupCredentials(
                   com.cloudbees.plugins.credentials.common.StandardUsernamePasswordCredentials.class,
                   jenkins.model.Jenkins.instance
                  )
def matchingCredentials = credentials.findResult { it.id == "harbor-credentials" ? it : null }
def HarborUser = "${matchingCredentials.username}".toString()
def HarborPassword = "${matchingCredentials.password}".toString()

node ()  {
    wrap([$class: 'BuildUser']) {
    	wrap([$class: 'MaskPasswordsBuildWrapper']) {
           wrap([$class: 'TimestamperBuildWrapper'] ) {
               wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) {
                  step([$class: 'WsCleanup'])
                   stage('Clone repositories') {
                         checkout scm
                       }

                   stage('Build docker image and run tests') {
                       this.buildImage(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword)
                        }

                   stage('Scan docker image') {
                       this.scanImage(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword)
                        }

                   stage('SonarQube Analysis') {
                       def scannerHome = tool 'sonar';
                       withSonarQubeEnv('sonar') {
                           sh "${scannerHome}/bin/sonar-scanner"
                          }
                        }

                   stage('Push docker image to harbor') {
                       this.pushImages(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword)
                        }

                   stage('Publish reports') {
                       publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, includes: 'report.html', keepAll: true, reportDir: '.', reportFiles: 'report.html', reportName: 'Trivy Scans', reportTitles: '', useWrapperFileDirectly: true])

}

                    }
                }
            }     
         }
    }
def removeAutodeleteImages() {
    this.runDocker('image prune -a -f --filter "label=autodelete=true"')
    echo 'removed autodelete images'
}

def withDockerCleanup(f) {
    try {
        this.removeAutodeleteImages()
        f()
    } finally {
        this.removeAutodeleteImages()
        this.runDocker('images')
    }
}


def runDocker(command) {
    sh("sudo docker ${command}")
}

def buildImage(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword) {
        // def getImagesCmd = "curl -u ${HarborUser}:${HarborPassword} -X GET 'http://${HarborUrl}:8081/service/rest/v1/search?repository=${HarborRegistry}&name=${HarborRegistry}/${RepositoryName}'"
        // def findLastSemanticVerCmd = "jq -r -c --raw-output '.items[].version'  | sort"
        // def incVersionCmd = 'perl -pe \'s/^((\\d+\\.)*)(\\d+)(.*)$/$1.($3+1).$4/e\''
        // def fullCmd = "${getImagesCmd} | ${findLastSemanticVerCmd} | ${incVersionCmd}"
        // imageVersion = sh(returnStdout: true, script: fullCmd).trim()
        // if (!imageVersion) {
        //     imageVersion = '1.0.0'
        // }
        // echo 'Next Image Version: ' + imageVersion
        def gitHash=sh (returnStdout: true, script: "git rev-parse HEAD").trim()
        def dateFormat = new SimpleDateFormat("yyyy-MM-dd_HH_mm_ss")
        def date = new Date()
        def buildDate = (dateFormat.format(date)) 
        // sh("docker build -t ${HarborUrl}/${HarborRegistry}/${RepositoryName}:1.0.${BUILD_ID}    .")
        sh("docker build -t ${HarborUrl}/library/${RepositoryName}:1.0.${BUILD_ID}    .")
}

def scanImage(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword) {
        echo "Scanning ${RepositoryName}"
        env.RepositoryName = "${RepositoryName}"
        env.HarborUser = "${HarborUser}"
        env.HarborPassword = "${HarborPassword}"
        env.HarborUrl = "${HarborUrl}"
        env.HarborRegistry = "${HarborRegistry}"
        sh label: '', script: '''#!/usr/bin/env bash
                                 export DOCKER_HOST=unix:///Users/gauravkothiyal/.docker/run/docker.sock 
                                 trivy image   --dependency-tree   -s MEDIUM,HIGH,CRITICAL  --ignore-unfixed --exit-code 0   --format template --template "@html.tpl" -o report.html \${HarborUrl}/library/\${RepositoryName}:1.0.\${BUILD_ID}'''
}

def pushImages(RepositoryName, HarborUrl, HarborRegistry, HarborUser, HarborPassword) {
        echo "Pushing ${RepositoryName}"
        env.RepositoryName = "${RepositoryName}"
        env.HarborUser = "${HarborUser}"
        env.HarborPassword = "${HarborPassword}"
        env.HarborUrl = "${HarborUrl}"
        env.HarborRegistry = "${HarborRegistry}"
        sh label: '', script: '''#!/usr/bin/env bash
set -x
                                 docker login -u \${HarborUser} -p \${HarborPassword} \${HarborUrl}
                                 docker push \${HarborUrl}/library/\${RepositoryName}:1.0.\${BUILD_ID}
                                 docker rmi \${HarborUrl}/library/\${RepositoryName}:1.0.\${BUILD_ID}'''
    }
