#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}

def pipelineMetadata = [
    pipelineName: 'rpminspect',
    pipelineDescription: 'Run rpminspect on RPM builds',
    testCategory: 'static-analysis',
    testType: 'rpminspect',
    maintainer: 'Fedora CI',
    docs: 'https://rpminspect.readthedocs.io/en/latest/',
    contact: [
        irc: '#fedora-ci',
        email: 'ci@lists.fedoraproject.org'
    ],
]
def artifactId
def testingFarmRequestId
def testingFarmResult
def repoUrlAndRef
def pipelineRepoUrlAndRef
def hook
def runUrl
def requestPayload


pipeline {

    agent none

    libraries {
        lib("fedora-pipeline-library@${env.PIPELINE_LIBRARY_VERSION}")
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: env.DEFAULT_DAYS_TO_KEEP_LOGS, artifactNumToKeepStr: env.DEFAULT_ARTIFACTS_TO_KEEP))
        timeout(time: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
        string(name: 'DIST_GIT_BRANCH', defaultValue: '', trim: true, description: "Dist-git branch associated with the provided ARTIFACT_ID")
    }

    environment {
        TESTING_FARM_API_KEY = credentials('testing-farm-api-key')
    }

    stages {
        stage('Prepare') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    artifactId = params.ARTIFACT_ID
                    setBuildNameFromArtifactId(artifactId: artifactId, profile: params.DIST_GIT_BRANCH)

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }

                    if (!params.DIST_GIT_BRANCH) {
                        abort('DIST_GIT_BRANCH is missing')
                    }

                    checkout scm
                    pipelineRepoUrlAndRef = [url: "${getGitUrl()}", ref: "${getGitRef()}"]
                }
                sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Schedule Test') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    requestPayload = [
                        api_key: "${env.TESTING_FARM_API_KEY}",
                        test: [
                            tmt: pipelineRepoUrlAndRef,
                        ],
                        environments: [
                            [
                                arch: "x86_64",
                                variables: [
                                    KOJI_TASK_ID: "${getIdFromArtifactId(artifactId: artifactId)}",
                                ],
                                tmt: [
                                    context: [
                                        "dist-git-branch": params.DIST_GIT_BRANCH,
                                    ]
                                ]
                            ]
                        ],
                    ]
                    hook = registerWebhook()
                    requestPayload['notification'] = ['webhook': [url: hook.getURL()]]

                    def response = submitTestingFarmRequest(payloadMap: requestPayload)
                    testingFarmRequestId = response['id']
                    runUrl = "${FEDORA_CI_TESTING_FARM_ARTIFACTS_URL}/${testingFarmRequestId}"
                }
                sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
            }
        }

        stage('Wait for Test Results') {
            agent none
            steps {
                script {
                    def response = waitForTestingFarm(requestId: testingFarmRequestId, hook: hook)
                    testingFarmResult = response.apiResponse
                }
            }
        }
    }

    post {
        always {
            evaluateTestingFarmResults(testingFarmResult)
        }
        aborted {
            script {
                if (isTimeoutAborted(timeout: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')) {
                    sendMessage(type: 'error', artifactId: artifactId, errorReason: 'Timeout has been exceeded, pipeline aborted.', pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
                }
            }
        }
        success {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
        failure {
            sendMessage(type: 'error', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
        unstable {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
    }
}
