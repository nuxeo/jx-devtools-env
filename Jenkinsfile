/*
 * (C) Copyright 2019 Nuxeo (http://nuxeo.com/) and others.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 * Contributors:
 *     Antoine Taillefer <ataillefer@nuxeo.com>
 */
 properties([
  [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/nuxeo/jx-devtools-env/']
])

void setGitHubBuildStatus(String context, String message, String state) {
  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/nuxeo/jx-devtools-env'],
    contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: context],
    statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'AnyBuildResult', message: message, state: state]]],
  ])
}

String getReleaseVersion() {
  return sh(returnStdout: true, script: 'jx-release-version')
}

String getJenkinsImageTag() {
  return sh(returnStdout: true, script: "grep ImageTag values.yaml | awk '{print \$2}'")
}

pipeline {
  agent {
    label 'jenkins-jx-base'
  }
  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(daysToKeepStr: '60', numToKeepStr: '60', artifactNumToKeepStr: '1'))
  }  
  environment {
    SERVICE_ACCOUNT = 'jenkins'
  }
  stages {
    stage('Build and push snapshot') {
      steps {
        setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X devtools', 'PENDING')
        container('jx-base') {
          echo "Upgrade Jenkins X devtools using jenkins image tag ${getJenkinsImageTag()}"
          script {
            // get the existing docker registry auth
            def dockerRegistryAuth = sh(
              script: """
                kubectl get secret jenkins-docker-cfg -o go-template='{{index .data "config.json"}}' | base64 --decode | sed -n '/"auth"/p' | awk -F'"' '{print \$4}'
              """,
              returnStdout: true
            ).trim();
            withEnv(["PUBLIC_DOCKER_REGISTRY_AUTH=${dockerRegistryAuth}"]) {
              sh """
              # initialize Helm without installing Tiller
              helm init --client-only --service-account ${SERVICE_ACCOUNT}

              # add local chart repository
              helm repo add jenkins-x http://chartmuseum.jenkins-x.io

              # replace env vars in values.yaml: DOCKER_REGISTRY, PUBLIC_DOCKER_REGISTRY_AUTH
              envsubst < values.yaml > myvalues.yaml

              # upgrade Jenkins X platform
              jx upgrade platform --namespace=devtools \
                --local-cloud-environment \
                --always-upgrade \
                --cleanup-temp-files=true \
                --version=2.0.1547 \
                --batch-mode
            """
            }
          }

          script {
            // get jenkins pod
            def jenkinsPod = sh(
              script: "kubectl get pod -l app=jenkins -o jsonpath='{..metadata.name}'",
              returnStdout: true
            ).trim()
            if (jenkinsPod) {
              // delete jenkins pod to recreate it
              sh "kubectl delete pod ${jenkinsPod} --ignore-not-found=true"
              echo "Deleted pod ${jenkinsPod} to recreate it."
            } else {
              echo "No jenkins pod found, won't recreate it."
            }
          }
        }
      }
      post {
        success {
          setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X platform', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X platform', 'FAILURE')
        }
      }
    }
    stage('Perform Git release') {
      // TODO: skip if no changes since latest tag, to be able to manually run the pipeline on the master branch
      // in order to revert a bad "jx upgrade platform" launched by a PR. This would run "jx upgrade platform"
      // on the latest (stable) tag, aka master, without adding add an extra Git tag.
      when {
        branch 'master'
      }
      steps {
        setGitHubBuildStatus('git-release', 'Perform Git release', 'PENDING')
        container('jx-base') {
          withEnv(["VERSION=${getReleaseVersion()}"]) {
            sh """
              # ensure we're not on a detached head
              git checkout master

              # create the Git credentials
              jx step git credentials
              git config credential.helper store

              # Git tag
              jx step tag -v ${VERSION}

              # Git release
              jx step changelog -v v${VERSION}
            """
          }
        }
      }
      post {
        always {
          step([$class: 'JiraIssueUpdater', issueSelector: [$class: 'DefaultIssueSelector'], scm: scm])
        }
        success {
          setGitHubBuildStatus('git-release', 'Perform Git release', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('git-release', 'Perform Git release', 'FAILURE')
        }
      }
    }
  }
}
