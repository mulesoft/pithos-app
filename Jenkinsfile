#!/usr/bin/env groovy
def propagateParamsToEnv() {
  for (param in params) {
    if (env."${param.key}" == null) {
      env."${param.key}" = param.value
    }
  }
}

properties([
  disableConcurrentBuilds(),
  parameters([
    string(name: 'TAG',
           defaultValue: '',
           description: 'Git tag to build'),
    booleanParam(defaultValue: false,
           description: 'Run or skip robotest system wide tests.',
           name: 'RUN_ROBOTEST'),
    choice(choices: ["true", "false"].join("\n"),
           description: 'Destroy all VMs on success.',
           name: 'DESTROY_ON_SUCCESS'),
    choice(choices: ["true", "false"].join("\n"),
           description: 'Destroy all VMs on failure.',
           name: 'DESTROY_ON_FAILURE'),
    choice(choices: ["true", "false"].join("\n"),
           description: 'Abort all tests upon first failure.',
           name: 'FAIL_FAST'),
    choice(choices: ["gce"].join("\n"),
           description: 'Cloud provider to deploy to.',
           name: 'DEPLOY_TO'),
    string(name: 'PARALLEL_TESTS',
           defaultValue: '4',
           description: 'Number of parallel tests to run.'),
    string(name: 'REPEAT_TESTS',
           defaultValue: '1',
           description: 'How many times to repeat each test.'),
    string(name: 'ROBOTEST_VERSION',
           defaultValue: '2.2.1',
           description: 'Robotest tag to use.'),
    string(name: 'GRAVITY_VERSION',
           defaultValue: '7.0.31',
           description: 'gravity/tele binaries version'),
    string(name: 'CLUSTER_SSL_APP_VERSION',
           defaultValue: '0.8.7',
           description: 'cluster-ssl-app version'),
    string(name: 'INTERMEDIATE_RUNTIME_VERSION',
           defaultValue: '6.1.43',
           description: 'Version of runtime to upgrade with'),
    string(name: 'EXTRA_GRAVITY_OPTIONS',
           defaultValue: '',
           description: 'Gravity options to add when calling tele'),
    string(name: 'TELE_BUILD_EXTRA_OPTIONS',
           defaultValue: '',
           description: 'Extra options to add when calling tele build'),
    booleanParam(name: 'ADD_GRAVITY_VERSION',
                 defaultValue: false,
                 description: 'Appends "-${GRAVITY_VERSION}" to the tag to be published'),
    booleanParam(name: 'BUILD_GRAVITY_APP',
                 defaultValue: true,
                 description: 'Generate a Gravity App tarball'),
    booleanParam(name: 'BUILD_CLUSTER_IMAGE',
                 defaultValue: false,
                 description: 'Generate a Gravity Cluster Image(Self-sufficient tarball)'),
    string(name: 'S3_UPLOAD_PATH',
           defaultValue: '',
           description: 'S3 bucket and path to upload built application image. For example "builds.example.com/cluster-ssl-app".'),
    booleanParam(name: 'PUBLISH_APP_PACKAGE',
                 defaultValue: false,
                 description: 'Import application to S3 bucket'),
    string(name: "AWS_CREDENTIALS",
           defaultValue: '',
           description: 'Name of the AWS credentials to use'),
    ]),
])

node {
  workspace {
    stage('checkout') {
      print 'Running stage Checkout source'

      def branches
      if (params.TAG == '') { // No tag specified
        branches = scm.branches
      } else {
        branches = [[name: "refs/tags/${params.TAG}"]]
      }

      checkout([
        $class: 'GitSCM',
        branches: branches,
        doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
        extensions: [[$class: 'CloneOption', noTags: false, shallow: false]],
        submoduleCfg: [],
        userRemoteConfigs: scm.userRemoteConfigs,
      ])
    }
    stage('params') {
      echo "${params}"
      propagateParamsToEnv()
    }
    stage('clean') {
      sh "make clean"
    }

    APP_VERSION = sh(script: 'make what-version', returnStdout: true).trim()
    APP_VERSION = params.ADD_GRAVITY_VERSION ? "${APP_VERSION}-${GRAVITY_VERSION}" : APP_VERSION
    STATEDIR = "${pwd()}/state/${APP_VERSION}"
    BINARIES_DIR = "${pwd()}/bin"
    MAKE_ENV = [
      "STATEDIR=${STATEDIR}",
      "PATH+GRAVITY=${BINARIES_DIR}",
      "VERSION=${APP_VERSION}"
    ]

    stage('download gravity/tele binaries') {
      withEnv(MAKE_ENV + ["BINARIES_DIR=${BINARIES_DIR}"]) {
        sh 'make download-binaries'
      }
    }

    stage('populate state directory with gravity and cluster-ssl packages') {
      withEnv(MAKE_ENV + ["BINARIES_DIR=${BINARIES_DIR}"]) {
        sh 'tele logout && make install-dependent-packages'
      }
    }

    stage('build-app') {
      if (params.BUILD_CLUSTER_IMAGE) {
        withEnv(MAKE_ENV) {
          sh 'make build-app'
        }
      } else {
        echo 'skipped build of gravity cluster image'
      }
    }

    stage('build gravity app') {
      if (params.BUILD_GRAVITY_APP) {
        withEnv(MAKE_ENV) {
          sh 'make export'
        }
      } else {
        echo 'skipped build of gravity application'
      }
    }

    stage('upload application tarball to S3') {
      if (isProtectedBranch(env.TAG) && params.PUBLISH_APP_PACKAGE) {
        withCredentials([usernamePassword(credentialsId: "${AWS_CREDENTIALS}", usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          def s3Url = "s3://${S3_UPLOAD_PATH}/pithos-app:${APP_VERSION}.tar"
          sh "aws s3 cp --only-show-errors build/application.tar ${s3Url}"
        }
      }
    }

    stage('test') {
      if (params.RUN_ROBOTEST) {
        throttle(['robotest']) {
            withCredentials([
              [$class: 'FileBinding', credentialsId:'ROBOTEST_LOG_GOOGLE_APPLICATION_CREDENTIALS', variable: 'GOOGLE_APPLICATION_CREDENTIALS'],
              [$class: 'StringBinding', credentialsId: params.OPS_CENTER_CREDENTIALS, variable: 'API_KEY'],
              [$class: 'FileBinding', credentialsId:'OPS_SSH_KEY', variable: 'SSH_KEY'],
              [$class: 'FileBinding', credentialsId:'OPS_SSH_PUB', variable: 'SSH_PUB'],
            ]) {
              def TELE_STATE_DIR = "${pwd()}/state/${APP_VERSION}"
              sh """
              export PATH=\$(pwd)/bin:\${PATH}
              export EXTRA_GRAVITY_OPTIONS="--state-dir=${TELE_STATE_DIR}"
              make robotest-run-suite \
                AWS_KEYPAIR=ops \
                AWS_REGION=us-east-1 \
                ROBOTEST_VERSION=$ROBOTEST_VERSION"""
            }
        }
      } else {
        echo 'skipped system tests'
      }
    }
  }
}

def isProtectedBranch(branchOrTagName) {
  // check if tag or branch empty
  if (!branchOrTagName?.trim()) {
    return false
  }

  String[] protectedBranches = ['master', 'support/.*']

  return protectedBranches.any { protectedBranch ->
    if (branchOrTagName == protectedBranch) return true
    def status = sh(script: "git branch --all --contains=${branchOrTagName} | grep '[*[:space:]]*remotes/origin/${protectedBranch}\$'", returnStatus: true)
    return status == 0
  }
}

void workspace(Closure body) {
  timestamps {
    ws("${pwd()}-${BUILD_ID}") {
      body()
    }
  }
}
