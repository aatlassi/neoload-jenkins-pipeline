pipeline {
  agent none
  environment {
    NLW_TOKEN = credentials('NLW_TOKEN_FROM_SAAS')
  }
  options {
    disableConcurrentBuilds()
    disableResume()
  }
  stages {
    stage('Start NeoLoad Infrastructure') {
      agent { label 'master' }
      steps {
        sh 'docker network create neoload'
        sh 'docker-compose -f neoload/load-generators/docker-compose.yml up -d'
        stash includes: 'neoload/load-generators/lg.yaml', name: 'LG'
        stash includes: 'neoload/load-generators/docker-compose.yml', name: 'infra'
        stash includes: 'Jenkinsfile', name: 'Jenkinsfile'
      }
    }
    stage('API Tests') {
      agent {
        dockerfile {
          args '--user root -v /tmp:/tmp --network=neoload --name=docker-controller'
          dir 'neoload/controller'
        }
      }
      steps {
        git(branch: "main", url: 'https://github.com/aatlassi/neoload-as-code-demo.git')
        unstash 'LG'
        sh script: "NeoLoad -project '$WORKSPACE/default.yaml' -testResultName 'Petstore API (build ${BUILD_NUMBER})' -description 'Testing Load as Code' -launch 'Petstore API' -loadGenerators '$WORKSPACE/neoload/load-generators/lg.yaml' -nlweb -nlwebAPIURL https://neoload-api.saas.neotys.com/explore/ -nlwebToken ${NLW_TOKEN} -leaseServer nlweb -leaseLicense 10:1"
      }
    }
  }
  post {
    always {
      node('master') {
        unstash 'infra'
        unstash 'Jenkinsfile'
        sh 'docker-compose -f neoload/load-generators/docker-compose.yml down'
        sh 'docker network rm neoload'
        script {
          NEOLOAD_PROJECT_FILES = sh (
            script: "ls -F | grep -vE  'neoload|Jenkinsfile|*.bak' | tr '\n' ',' ; echo",
            returnStdout: true
          ).trim().replaceAll("/","/**")
          zip archive: true, dir: '', glob: "${NEOLOAD_PROJECT_FILES}", zipFile: 'neoload_as_code_demo.zip'
        }
        archiveArtifacts allowEmptyArchive: true, artifacts: 'results/**,Jenkinsfile,neoload/**,common/**,v1/**,default.yaml'
        sh 'docker volume prune -f'
        sh 'docker image prune -f'
        cleanWs()
      }
    }
  }
}
