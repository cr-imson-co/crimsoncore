#!groovy

def PYTHON_VERSION = '3.8'
pipeline {
  options {
    copyArtifactPermission 'cr.imson.co/lambda/deploy-pipeline'
    buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '3', daysToKeepStr: '', numToKeepStr: '')
    gitLabConnection('gitlab@cr.imson.co')
    gitlabBuilds(builds: ['jenkins'])
    disableConcurrentBuilds()
    timestamps()
  }
  post {
    failure {
      updateGitlabCommitStatus name: 'jenkins', state: 'failed'
    }
    unstable {
      updateGitlabCommitStatus name: 'jenkins', state: 'failed'
    }
    aborted {
      updateGitlabCommitStatus name: 'jenkins', state: 'canceled'
    }
    success {
      updateGitlabCommitStatus name: 'jenkins', state: 'success'
    }
    always {
      cleanWs()
    }
  }
  agent {
    docker {
      image "docker.cr.imson.co/python-lambda-builder:${PYTHON_VERSION}"
    }
  }
  environment {
    CI = 'true'
    LAYER_NAME = 'crimsoncore'
    LAYER_RUNTIME = "python${PYTHON_VERSION}"
    LAYER_DESCRIPTION = "CrimsonCore Lambda Layer for Python ${PYTHON_VERSION}"
    LAYER_LICENSE = 'MIT'
    PIP_DISABLE_PIP_VERSION_CHECK = '1'
  }
  stages {
    stage('Prepare') {
      steps {
        updateGitlabCommitStatus name: 'jenkins', state: 'running'
        sh 'python --version && pip --version'
      }
    }

    stage('Run pylint') {
      environment {
        HOME = "${env.WORKSPACE}"
      }
      steps {
        // tight coupling against boto3 requires us to install it and its deps in order to lint
        sh label: 'install dependencies for pylint run',
          script: "pip install --user --no-cache --progress-bar off -r ${env.WORKSPACE}/deps/boto3/requirements.txt"

        sh label: 'run pylint',
          script: "find ${env.WORKSPACE}/lib/${env.LAYER_NAME} -type f -iname '*.py' -print0 | xargs -0 python -m pylint"
      }
    }

    stage('Run tests') {
      environment {
        HOME = "${env.WORKSPACE}"
      }
      steps {
        sh label: 'install library to run tests',
          script: "pip install --user --no-cache --progress-bar off -e ${env.WORKSPACE}/lib"

        sh label: 'run unit tests',
          script: 'python -m unittest discover'
      }
    }

    stage('Build layer') {
      steps {
        sh label: 'create build directory',
          script: "mkdir -p ${env.WORKSPACE}/build/python/lib/python${PYTHON_VERSION}/site-packages/${env.LAYER_NAME}/"

        sh label: 'install package to build directory',
          script: """
            cp \
              ${env.WORKSPACE}/lib/${env.LAYER_NAME}/*.py \
              ${env.WORKSPACE}/build/python/lib/python${PYTHON_VERSION}/site-packages/${env.LAYER_NAME}/
          """.stripIndent()

        sh label: 'install boto3 and dependencies to build directory',
          script: """
            pip install \
              --no-cache \
              --progress-bar off \
              -r ${env.WORKSPACE}/deps/boto3/requirements.txt \
              -t ${env.WORKSPACE}/build/python/lib/python${PYTHON_VERSION}/site-packages/.
          """

        sh label: 'install pytz and dependencies to build directory',
          script: """
            pip install \
              --no-cache \
              --progress-bar off \
              -r ${env.WORKSPACE}/deps/pytz/requirements.txt \
              -t ${env.WORKSPACE}/build/python/lib/python${PYTHON_VERSION}/site-packages/.
          """

        dir("${env.WORKSPACE}/build/") {
          sh label: 'build layer zip',
            script: "zip -r ${env.LAYER_NAME}-lambda-layer.zip *"
        }
      }
    }

    stage('Archive artifacts') {
      when {
        branch 'master'
      }
      steps {
        archiveArtifacts "build/${env.LAYER_NAME}-lambda-layer.zip"

        build job: 'cr.imson.co/lambda/deploy-pipeline',
          parameters: [
            [$class: 'StringParameterValue', name: 'LAYER_NAME', value: env.LAYER_NAME],
            [$class: 'StringParameterValue', name: 'LAYER_DESCRIPTION', value: env.LAYER_DESCRIPTION],
            [$class: 'StringParameterValue', name: 'LAYER_LICENSE', value: env.LAYER_LICENSE],
            [$class: 'StringParameterValue', name: 'LAYER_RUNTIME', value: env.LAYER_RUNTIME],
            [$class: 'StringParameterValue', name: 'LAYER_ARTIFACT_NAME', value: "${env.LAYER_NAME}-lambda-layer.zip"],
            [$class: 'StringParameterValue', name: 'ORIGINAL_JOB', value: currentBuild.fullProjectName],
            [$class: 'StringParameterValue', name: 'ORIGINAL_BUILD_NUMBER', value: env.BUILD_NUMBER]
          ],
          wait: true
      }
    }
  }
}
