#!groovy

/*
 * Copyright © 2016, 2017 IBM Corp. All rights reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file
 * except in compliance with the License. You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed under the
 * License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
 * either express or implied. See the License for the specific language governing permissions
 * and limitations under the License.
 */

 // Get the IP of a docker container
def hostIp(container) {
  sh "docker inspect -f '{{.Node.IP}}' ${container.id} > hostIp"
  readFile('hostIp').trim()
}

def runTestsCalled(name) {
  // Define the matrix environments
  def CLOUDANT_ENV = ['DB_HTTP=https', 'DB_HOST=clientlibs-test.cloudant.com', 'DB_PORT=443', 'DB_IGNORE_COMPACTION=true', 'GRADLE_TARGET=cloudantServiceTest']
  def CONTAINER_ENV = ['DB_HTTP=http', 'DB_PORT=5984', 'DB_IGNORE_COMPACTION=false', 'GRADLE_TARGET=test']

  node {
    if (name == 'cloudant') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'clientlibs-test', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD']]) {
        runTests(CLOUDANT_ENV)
      }
    } else {
      def curlCreds = ''
      def createUsers = false
      def createReplicator = false
      switch(name) {
        case 'klaemo/couchdb:2.0.0':
          createUsers = true
          createReplicator = true
          containerPort = 5984
          break
        case 'couchdb:1.6.1':
          containerPort = 5984
          break
        case 'ibmcom/cloudant-developer':
          containerPort = 80
          createReplicator = true
          CONTAINER_ENV.add('DB_USER=admin')
          CONTAINER_ENV.add('DB_PASSWORD=pass')
          curlCreds = "admin:pass@"
          break
        default:
          error("Unknown test env ${name}")
      }
      docker.withServer(env.DOCKER_SWARM_URL) {
        docker.image(name).withRun("-p 5984:${containerPort}") {container ->
          dbHost = hostIp(container)
          CONTAINER_ENV.add("DB_HOST=${dbHost}")
          if (createReplicator) {
            sh "sleep 3 && curl -X PUT https://${curlCreds}${dbHost}:5984/_replicator"
            // --retry 3 --retry-connrefused would be preferable but is not yet
            // available in this version of curl
          }
          if (createUsers) {
            sh "curl -X PUT https://${curlCreds}${dbHost}:5984/_users"
          }
          runTests(CONTAINER_ENV)
        }
      }
    }
  }
}

def runTests(testEnv) {
  // Unstash the built content
  unstash name: 'built'

  //Set up the environment and run the tests
  withEnv(testEnv) {
    testCmd = './gradlew'
    if (env.DB_USER) {
      testCmd += ' -Dtest.couch.username=$DB_USER -Dtest.couch.password=$DB_PASSWORD'
    }
    testCmd += ' -Dtest.couch.host=$DB_HOST -Dtest.couch.port=$DB_PORT -Dtest.couch.http=$DB_HTTP $GRADLE_TARGET'
    try {
      sh testCmd
    } finally {
      junit '**/build/test-results/*.xml'
    }
  }
}

stage('Build') {
    // Checkout, build and assemble the source and doc
    node {
        checkout scm
        sh './gradlew clean assemble'
        stash name: 'built'
    }
}

stage('QA') {
    // Standard builds do Findbugs and test against Cloudant
    def axes = [
            Findbugs:
                    {
                        node {
                            unstash name: 'built'
                            // findBugs
                            try {
                                sh './gradlew -Dfindbugs.xml.report=true findbugsMain'
                            } finally {
                                step([$class: 'FindBugsPublisher', pattern: '**/build/reports/findbugs/*.xml'])
                            }
                        }
                    },
            Cloudant: {
                runTestsCalled('cloudant')
            }
    ]

    // For the master branch, add additional axes to the coverage matrix for Couch 1.6, 2.0
    // and Cloudant Local
    if (env.BRANCH_NAME == "master") {
        axes.putAll(
                Couch1_6: {
                    runTestsCalled('couchdb:1.6.1')
                },
                Couch2_0: {
                    runTestsCalled('klaemo/couchdb:2.0.0')
                },
                CloudantLocal: {
                    runTestsCalled('ibmcom/cloudant-developer')
                }
        )
    }

    // Run the required axes in parallel
    parallel(axes)
}

// Publish the master branch
stage('Publish') {
    if (env.BRANCH_NAME == "master") {
        node {
            checkout scm // re-checkout to be able to git tag
            unstash name: 'built'
            // read the version name and determine if it is a release build
            version = readFile('VERSION').trim()
            isReleaseVersion = !version.toUpperCase(Locale.ENGLISH).contains("SNAPSHOT")

            // Upload using the ossrh creds (upload destination logic is in build.gradle)
            withCredentials([usernamePassword(credentialsId: 'ossrh-creds', passwordVariable: 'OSSRH_PASSWORD', usernameVariable: 'OSSRH_USER'), usernamePassword(credentialsId: 'signing-creds', passwordVariable: 'KEY_PASSWORD', usernameVariable: 'KEY_ID'), file(credentialsId: 'signing-key', variable: 'SIGNING_FILE')]) {
                sh './gradlew -Dsigning.keyId=$KEY_ID -Dsigning.password=$KEY_PASSWORD -Dsigning.secretKeyRingFile=$SIGNING_FILE -DossrhUsername=$OSSRH_USER -DossrhPassword=$OSSRH_PASSWORD upload'
            }

            // if it is a release build then do the git tagging
            if (isReleaseVersion) {

                // Read the CHANGES.md to get the tag message
                changes = """"""
                changes += readFile('CHANGES.md')
                tagMessage = """"""
                for (line in changes.readLines()) {
                    if (!"".equals(line)) {
                        // append the line to the tagMessage
                        tagMessage = "${tagMessage}${line}\n"
                    } else {
                        break
                    }
                }

                // Use git to tag the release at the version
                try {
                    // Awkward workaround until resolution of https://issues.jenkins-ci.org/browse/JENKINS-28335
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github-token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
                        sh "git config user.email \"nomail@hursley.ibm.com\""
                        sh "git config user.name \"Jenkins CI\""
                        sh "git config credential.username ${env.GIT_USERNAME}"
                        sh "git config credential.helper '!echo password=\$GIT_PASSWORD; echo'"
                        sh "git tag -a ${version} -m '${tagMessage}'"
                        sh "git push origin ${version}"
                    }
                } finally {
                    sh "git config --unset credential.username"
                    sh "git config --unset credential.helper"
                }
            }
        }
    }
}
