// todo get these podTemplates from templates folder?
def sonarqubePodLabel = "eagle-api-${UUID.randomUUID().toString()}"
// podTemplate(label: sonarqubePodLabel, name: sonarqubePodLabel, serviceAccount: 'jenkins', cloud: 'openshift', containers: [])

@NonCPS
import groovy.json.JsonOutput
import groovy.json.JsonSlurper
/*
 * Sends a rocket chat notification
 */
def notifyRocketChat(text, url) {
    def rocketChatURL = url
    def payload = JsonOutput.toJson([
      "username":"Jenkins",
      "icon_url":"https://wiki.jenkins.io/download/attachments/2916393/headshot.png",
      "text": text
    ])

    sh("curl -X POST -H 'Content-Type: application/json' --data \'${payload}\' ${rocketChatURL}")
}

/*
 * takes in a sonarqube status json payload
 * and returns the status string
 */
def sonarGetStatus (jsonPayload) {
  def jsonSlurper = new JsonSlurper()
  return jsonSlurper.parseText(jsonPayload).projectStatus.status
}

/*
 * Updates the global pastBuilds array: it will iterate recursively
 * and add all the builds prior to the current one that had a result
 * different than 'SUCCESS'.
 */
def buildsSinceLastSuccess(previousBuild, build) {
  if ((build != null) && (build.result != 'SUCCESS')) {
    pastBuilds.add(build)
    buildsSinceLastSuccess(pastBuilds, build.getPreviousBuild())
  }
}

/*
 * Generates a string containing all the commit messages from
 * the builds in pastBuilds.
 */
@NonCPS
def getChangeLog(pastBuilds) {
  def log = ""
  for (int x = 0; x < pastBuilds.size(); x++) {
    for (int i = 0; i < pastBuilds[x].changeSets.size(); i++) {
      def entries = pastBuilds[x].changeSets[i].items
      for (int j = 0; j < entries.length; j++) {
        def entry = entries[j]
        log += "* ${entry.msg} by ${entry.author} \n"
      }
    }
  }
  return log;
}

def nodejsTester () {
  openshift.withCluster() {
    openshift.withProject() {
      podTemplate(label: 'node-tester', name: 'node-tester', serviceAccount: 'jenkins', cloud: 'openshift', containers: [
        containerTemplate(
          name: 'jnlp',
          image: 'registry.access.redhat.com/openshift3/jenkins-agent-nodejs-8-rhel7',
          resourceRequestCpu: '500m',
          resourceLimitCpu: '800m',
          resourceRequestMemory: '2Gi',
          resourceLimitMemory: '4Gi',
          workingDir: '/tmp',
          command: '',
        )
      ]) {
        node("node-tester") {
          checkout scm
          try {
            sh 'npm run tests'
          } finally {
            echo "Unit Tests Passed"
          }
        }
      }
      return true
    }
  }
}

// todo templates can be pulled from a repository, instead of declared here
def nodejsSonarqube () {
  openshift.withCluster() {
    openshift.withProject() {
      podTemplate(label: 'node-sonarqube', name: 'node-sonarqube', serviceAccount: 'jenkins', cloud: 'openshift', containers: [
        containerTemplate(
          name: 'jnlp',
          image: 'registry.access.redhat.com/openshift3/jenkins-agent-nodejs-8-rhel7',
          resourceRequestCpu: '500m',
          resourceLimitCpu: '1000m',
          resourceRequestMemory: '2Gi',
          resourceLimitMemory: '4Gi',
          workingDir: '/tmp',
          command: '',
          args: '${computer.jnlpmac} ${computer.name}',
        )
      ]) {
        node("node-sonarqube") {
          checkout scm
          dir('sonar-runner') {
            try {
              // run scan
              sh("oc extract secret/sonarqube-secrets --to=${env.WORKSPACE}/sonar-runner --confirm")
              SONARQUBE_URL = sh(returnStdout: true, script: 'cat sonarqube-route-url')

              sh "npm install typescript"
              sh returnStdout: true, script: "./gradlew sonarqube -Dsonar.host.url=${SONARQUBE_URL} -Dsonar. -Dsonar.verbose=true --stacktrace --info"

              // check if sonarqube passed
              sh("oc extract secret/sonarqube-status-urls --to=${env.WORKSPACE}/sonar-runner --confirm")
              SONARQUBE_STATUS_URL = sh(returnStdout: true, script: 'cat sonarqube-status-api')

              SONARQUBE_STATUS_JSON = sh(returnStdout: true, script: "curl -w '%{http_code}' '${SONARQUBE_STATUS_URL}'")
              SONARQUBE_STATUS = sonarGetStatus (SONARQUBE_STATUS_JSON)

              if ( "${SONARQUBE_STATUS}" == "ERROR") {
                echo "Scan Failed"

                notifyRocketChat(
                  "@all The latest build of eagle-api seems to be broken. \n Error: \n Sonarqube scan failed",
                  ROCKET_QA_WEBHOOK
                )

                currentBuild.result = 'FAILURE'
              } else {
                echo "Scan Passed"
              }

            } catch (error) {
              notifyRocketChat(
                "@all The latest build of eagle-api seems to be broken. \n Error: \n ${error}",
                ROCKET_QA_WEBHOOK
              )
              throw error
            } finally {
              echo "Scan Complete"
            }
          }
        }
      }
      return true
    }
  }
}

def CHANGELOG = "No new changes"
def IMAGE_HASH = "latest"

pipeline {
  agent any
  options {
    disableResume()
  }
  stages {
    stage('Parallel Build Steps') {
      failFast true
      parallel {
        stage('Build') {
          agent any
          steps {
            script {
              pastBuilds = []
              buildsSinceLastSuccess(pastBuilds, currentBuild);
              CHANGELOG = getChangeLog(pastBuilds);

              echo ">>>>>>Changelog: \n ${CHANGELOG}"

              try {
                sh("oc extract secret/rocket-chat-secrets --to=${env.WORKSPACE} --confirm")
                ROCKET_DEPLOY_WEBHOOK = sh(returnStdout: true, script: 'cat rocket-deploy-webhook')
                ROCKET_QA_WEBHOOK = sh(returnStdout: true, script: 'cat rocket-qa-webhook')

                echo "Building eagle-api develop branch"
                openshiftBuild bldCfg: 'eagle-api', showBuildLogs: 'true'
                echo "Build done"

                echo ">>> Get Image Hash"
                // Don't tag with BUILD_ID so the pruner can do it's job; it won't delete tagged images.
                // Tag the images for deployment based on the image's hash
                IMAGE_HASH = sh (
                  script: """oc get istag eagle-api:latest -o template --template=\"{{.image.dockerImageReference}}\"|awk -F \":\" \'{print \$3}\'""",
                  returnStdout: true).trim()
                echo ">> IMAGE_HASH: ${IMAGE_HASH}"
              } catch (error) {
                notifyRocketChat(
                  "@all The latest build of eagle-api seems to be broken. \n Error: \n ${error}",
                  ROCKET_QA_WEBHOOK
                )
                throw error
              }
            }
          }
        }

        // stage('Unit Tests') {
        //   steps {
        //     script {
        //       echo "Running Unit Tests"
        //       def result = nodejsTester()
        //     }
        //   }
        // }

        stage('Sonarqube') {
          steps {
            script {
              echo "Running Sonarqube"
              def result = nodejsSonarqube()
            }
          }
        }
      }
    }

    stage('Deploy to dev'){
      steps {
        script {
          try {
            echo "Deploying to dev..."
            openshiftTag destStream: 'eagle-api', verbose: 'false', destTag: 'dev', srcStream: 'eagle-api', srcTag: "${IMAGE_HASH}"
            sleep 5
            // todo eagle-test? what depCfg?
            openshiftVerifyDeployment depCfg: 'eagle-api', namespace: 'esm-dev', replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false', waitTime: 600000
            echo ">>>> Deployment Complete"

            notifyRocketChat(
              "@all A new version of eagle-api is now in Dev. \n Changes: \n ${CHANGELOG}",
              ROCKET_DEPLOY_WEBHOOK
            )

            notifyRocketChat(
              "@all A new version of eagle-api is now in Dev and ready for QA. \n Changes to Dev: \n ${CHANGELOG}",
              ROCKET_QA_WEBHOOK
            )
          } catch (error) {
            notifyRocketChat(
              "@all The latest deployment of eagle-api to Dev seems to have failed\n Error: \n ${error.message}",
              ROCKET_DEPLOY_WEBHOOK
            )
            currentBuild.result = "FAILURE"
            throw new Exception("Deploy failed")
          }
        }
      }
    }
  }
}