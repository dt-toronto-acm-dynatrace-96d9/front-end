pipeline {
  agent {
    label 'nodejs'
  }
  environment {
    APP_NAME = "front-end"
    VERSION = readFile('version').trim()
    ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/sockshop-registry/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}:${env.VERSION}"
  }
  stages {
    stage('Node build') {
      steps {
        checkout scm
        container('nodejs') {
          sh 'npm install'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'registry-creds', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "docker login --username=anything --password=${TOKEN} ${env.DOCKER_REGISTRY_URL}:5000"
            sh "docker tag ${env.TAG_DEV} ${env.TAG_DEV}"
            sh "docker push ${env.TAG_DEV}"
          }
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "sed -i 's#image: .*#image: ${env.TAG_DEV}#' manifest/front-end.yml"
          sh "kubectl -n dev apply -f manifest/front-end.yml"
        }
      }
    }
    stage('DT Deploy Event') {
        when {
            expression {
            return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
        }
        steps {
          container("curl") {
            // send custom deployment event to Dynatrace
            sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"dev\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${env.VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
          }
        }
    }
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
          sh "docker push ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}")
          ]
      }
    }  
  }
}
