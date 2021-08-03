pipeline {
  agent any
  environment {
    PRE_PROD_INVENTORY = "aws_preprod"
    STAGING_INVENTORY = "aws_staging"
    DEVELOPMENT_INVENTORY = "old_staging"
    PRE_PRODUCTION_BRANCH = /^release\/.*/
    STAGING_BRANCH = "integration"
    DEV_BRANCH = "cicd_dev"
    INVENTORY_NAME = """${
                BRANCH_NAME == DEV_BRANCH ? DEVELOPMENT_INVENTORY : 
                    BRANCH_NAME ==~ PRE_PRODUCTION_BRANCH ? PRE_PROD_INVENTORY : 
                        BRANCH_NAME == STAGING_BRANCH ? STAGING_INVENTORY : 'Unknown'
            }"""
    BUILD_URL = """${
                BRANCH_NAME == DEV_BRANCH ? 'https://staging-new.classicrummy.com/rummy-lite/' : 
                    BRANCH_NAME ==~ PRE_PRODUCTION_BRANCH ? 'https://preprod-new.classicrummy.com/rummy-lite/' : 
                        BRANCH_NAME == STAGING_BRANCH ? 'https://staging-new.classicrummy.com/rummy-lite/' : 'Unknown'
            }"""
  }
  stages {
      stage('Validate branch') {
        when {
            equals expected: 'Unknown',
            actual: env.INVENTORY_NAME
        }
        steps {
            script {
                    currentBuild.result = "ABORTED"
                    error "Aborting build as the branch not part of plan"
                    }
        }
      }
        stage ('Install RummyLite Dependencies') {
            steps {
                sh """
                  sudo n stable
                  sudo npm i
                   """
            }
        }
        stage ('Build RummyLite') {
            steps {
                sh """
                  ng build --prod --deploy-url=${BUILD_URL} --base-href=/rummy-lite/ --rebase-root-relative-css-urls
                 scp -r src/assets dist/rummy-lite/
                   """
            }
        }
        stage ('Change node modules permission') {
            steps {
                sh """
                  sudo chmod 777 -R node_modules
                   """
            }
        }
        stage('Prepare Artifact') {
            steps {
                echo "The branch: ${BRANCH_NAME} and the build ${BUILD_NUMBER}"
                sh """
                    tar -zcvf rummylite_${INVENTORY_NAME}_${BUILD_NUMBER}.tar.gz dist/rummy-lite/.
                    tar -zcvf rummylite_latest.tar.gz dist/rummy-lite/.
                """
            }
        }
        stage('Push to Artifactory') {
            steps {
                echo 'we will push the tar file to artifactory'
                rtServer (
                    id: 'jfrog',
                    url: 'http://10.16.51.18:8082/artifactory',
                    credentialsId: 'jfrog_login'
                )
                rtUpload (
                    serverId: 'jfrog',
                    spec: """{
                      "files": [
                    {
                        "pattern": "rummylite*.tar.gz",
                        "target": "rummylite/",
                        "props": "type=tar.gz;status=ready"
                    }
                        ]
                    }""",
                    buildName: 'rummylite',
                    buildNumber: '${BUILD_NUMBER}'
                )
            }
        }
        stage('Deploy') {
            steps {
                sh """
                  sudo ansible-playbook ansible.yml --extra-vars "deployment_host=${INVENTORY_NAME}"
                   """
            }
        }
    }
    post {
    aborted {
      emailext attachLog: true,
      mimeType: 'text/html',
      body: '${FILE, path="/var/lib/jenkins/mailtemplate/aborted.html"}',
      compressLog: true,
      recipientProviders: [developers(), requestor()],
      subject: '${DEFAULT_SUBJECT}',
      to: '$DEFAULT_RECIPIENTS'
    }
    failure {
      emailext attachLog: true,
      mimeType: 'text/html',
      body: '${FILE, path="/var/lib/jenkins/mailtemplate/failure.html"}',
      compressLog: true,
      recipientProviders: [developers(), requestor()],
      subject: '${DEFAULT_SUBJECT}',
      to: '$DEFAULT_RECIPIENTS'
    }
    success {
      emailext attachLog: true,
      mimeType: 'text/html',
      body: '${FILE, path="/var/lib/jenkins/mailtemplate/success.html"}',
      compressLog: true,
      recipientProviders: [developers(), requestor()],
      subject: '${DEFAULT_SUBJECT}',
      to: '$DEFAULT_RECIPIENTS'
    }
  }
}
