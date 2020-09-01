def isSnapshot
pipeline {
  agent {
    kubernetes {
      defaultContainer 'maven'
      inheritFrom 'default'
      yamlMergeStrategy merge()
      yaml """
apiVersion: v1
kind: Pod
metadata:
 labels:
   deployPipeline: ethos
spec:
  containers:
  - name: maven
    image: maven:3.6.3-jdk-8
    command:
    - cat
    tty: true
    imagePullPolicy: IfNotPresent
    envFrom:
      - configMapRef:
          name: proxy-environment-variables
      """
    }
  }
  options {
    buildDiscarder(
      logRotator(
        numToKeepStr: '50'
      )
    )
    timeout(time: 60, unit: 'MINUTES') // This will be increased when we have the ability to trigger/kill jobs ourselves
    timestamps()
  }
  parameters {
    choice(name: 'PROJECT_FILTER', choices: ['', 'firehose-record-transformer'], description: 'Targets a single module to build; specifying blank will build all modules.')
    booleanParam(name: 'PUBLISH_ARTIFACTS', defaultValue: false, description: 'Determines whether or not the artifacts will be packaged and deployed into Artifactory.')
    booleanParam(name: 'PUBLISH_DOCUMENTATION', defaultValue: false, description: 'Determines whether or not the documentation generated for each artifact will be published into Artifactory.')
  }
  stages {
    stage('Initialize') {
      steps {
        script {
          // If we are not publishing an artifact, then treat this as a snapshot build.
          isSnapshot = !params.PUBLISH_ARTIFACTS

          // Log the values for troubleshooting support.
          echo "Interpreted parameters used within the build:"
          echo "isSnapshot: ${isSnapshot}"
        }
      }
    }
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Compile') {
      steps {
        runMavenCommand("clean validate compile -DskipTests")
      }
    }
    stage('Test') {
      steps {
        runMavenCommand("test")
      }
    }
    stage('Package') {
      when {
        expression { params.PUBLISH_ARTIFACTS }
      }
      steps {
        runMavenCommand("package verify")
      }
    }
    stage('Documentation') {
      when {
        expression { params.PUBLISH_DOCUMENTATION }
      }
      steps {
        runMavenCommand("site")
      }
    }
    stage('Publish') {
      when {
        expression { params.PUBLISH_ARTIFACTS }
      }
      steps {
        runMavenCommand("deploy")
      }
    }
  }
}

def runMavenCommand(mavenCommand) {
  // Provides the necessary credentials to Artifactory for download/deploy purposes.
  withCredentials([usernamePassword(credentialsId: 'SvcAcct-artifactory', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
    // Run the appropriate command based on whether or not we are applying a project level filter.
    if(params.PROJECT_FILTER == "") {
      sh "mvn --batch-mode -V -U -e -s build/maven/settings.xml ${mavenCommand} -DproxySet=true -Dhttps.proxyHost=proxy.cdp-devl.us-east-1.edp.awscloud.worldpay.com -Dhttps.proxyPort=3128 -Dserver.username=${USERNAME} -Dserver.password=${PASSWORD}"
    } else {
      sh "mvn --batch-mode -V -U -e -s build/maven/settings.xml -pl ${params.PROJECT_FILTER} ${mavenCommand} -DproxySet=true -Dhttps.proxyHost=proxy.cdp-devl.us-east-1.edp.awscloud.worldpay.com -Dhttps.proxyPort=3128 -Dserver.username=${USERNAME} -Dserver.password=${PASSWORD}"
    }
  }
}