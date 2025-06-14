pipeline {
  agent any
  triggers {
    githubPush()
  }
  environment {
        AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
        LS_ES_EA_API = credentials('logstash_ea-demo')
  }
  stages {
    stage("Config validation") {
        parallel {
          stage("Config validation. - Demo Pipeline") {
                steps {
                  echo "Testing ..."
                  wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['AWS_ACCESS_KEY_ID'], var: 'SECRET'], [password: env['AWS_SECRET_ACCESS_KEY'], var: 'SECRET']]]) {
                    timeout(time: 30, unit: 'SECONDS') {
                        sh '''
                          /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
                        '''
                    }
                    // timeout(time: 3, unit: 'MINUTES') {
                    //     sh '''
                    //       /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
                    //     '''
                    // }
                    timeout(time: 45, unit: 'SECONDS') {
                      sh '''
                        echo "${AWS_SECRET_ACCESS_KEY}" | /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash add VAULT_SECRET
                      '''
                    }
                  }
                  timeout(time: 60, unit: 'SECONDS') {
                    sh '''#!/bin/bash
                    mkdir -p /tmp/pipeline_deployment
                    cp sample_pipeline-001.conf /tmp/pipeline_deployment && cat /tmp/pipeline_deployment/sample_pipeline-001.conf
                    if /usr/share/logstash/bin/logstash --path.settings /tmp/logstash -t -f /tmp/pipeline_deployment/sample_pipeline-001.conf | grep "Configuration OK"; then 
                      echo "Syntax OK"
                      exit 0
                    else
                      echo "Syntax Error"
                      exit 1 
                    fi
                    '''
                  }
                  post {
                    always {
                      timeout(time: 30, unit: 'SECONDS') {
                        sh '''
                        /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
                        '''
                      }
                    }
                  }
                }
              }
              stage("Config validation. - Elastic Agent Pipeline") {
                steps {
                  echo "Testing ..."
                  wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['LS_ES_EA_API'], var: 'SECRET']]]) {
                    timeout(time: 30, unit: 'SECONDS') {
                      sh '''
                        /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove ES_API_SECRET || true
                      '''
                    }
                    timeout(time: 45, unit: 'SECONDS') {
                      sh '''
                        echo "${LS_ES_EA_API}" | /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash add ES_API_SECRET
                      '''
                    }
                    
                  }
                  timeout(time: 60, unit: 'SECONDS') {
                    sh '''#!/bin/bash
                    mkdir -p /tmp/pipeline_deployment
                    cp sample_pipeline-ea.conf /tmp/pipeline_deployment && cat /tmp/pipeline_deployment/sample_pipeline-ea.conf
                    if /usr/share/logstash/bin/logstash --path.settings /tmp/logstash -t -f /tmp/pipeline_deployment/sample_pipeline-ea.conf | grep "Configuration OK"; then 
                      echo "Syntax OK"
                      exit 0
                    else
                      echo "Syntax Error"
                      exit 1 
                    fi
                    '''
                  }
                  post {
                    always {
                      timeout(time: 30, unit: 'SECONDS') {
                        sh '''
                        /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove ES_API_SECRET || true
                        '''
                      }
                    }
                  }
                }
              }
        }
    }
    stage('Update Local LS Keystore update') {
      steps {
          echo "Updating Keystore ..."
          wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['AWS_ACCESS_KEY_ID'], var: 'SECRET'], [password: env['AWS_SECRET_ACCESS_KEY'], var: 'SECRET']]]) {
            timeout(time: 30, unit: 'SECONDS') {
              sh '''
                /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash remove VAULT_SECRET || true
              '''
            }
            timeout(time: 30, unit: 'SECONDS') {
              sh '''
                echo "${AWS_SECRET_ACCESS_KEY}" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add VAULT_SECRET
              '''
            }
            timeout(time: 30, unit: 'SECONDS') {
              sh '''
              /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash remove ES_API_SECRET || true
              '''
            }
            timeout(time: 30, unit: 'SECONDS') {
              sh '''
                echo "${LS_ES_EA_API}" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add ES_API_SECRET
              '''
            }
          }
      }
    }
    stage('Deploy') {
      steps {
        retry(2) {
          echo "Deploying ... demo-jenkins-with_secret"
          script {
            def fileContent = readFile file: '/tmp/pipeline_deployment/sample_pipeline-001.conf'
            // Try to replace double quotes with single quotes
            fileContent = fileContent.replaceAll('"', "'")
            fileContent = fileContent.replaceAll('\r', '\\\\r')
            fileContent = fileContent.replaceAll('\n', '\\\\n')
            env.textData = fileContent
          }
          echo "${env.textData}"
          httpRequest httpMode: 'PUT', url: 'https://apm-dev-ac.es.us-east-2.aws.elastic-cloud.com/_logstash/pipeline/demo-jenkins-with_secret',
          acceptType: 'APPLICATION_JSON',
          contentType: 'APPLICATION_JSON',
          authentication: 'o11y_es_ingest',
          requestBody: """{
            "description": "", 
            "last_modified": "2025-06-04T02:50:51.250Z",
            "pipeline_metadata": {
              "type": "logstash_pipeline",
              "version": "1"
            },
            "username": "O11y-cicd-user",
            "pipeline": "${env.textData}",
            "pipeline_settings": {
              "pipeline.workers": 1,
              "pipeline.batch.size": 125,
              "pipeline.batch.delay": 50,
              "queue.type": "memory",
              "queue.max_bytes": "1gb",
              "queue.checkpoint.writes": 1024
            }
          }
          """
        }
      }
    }
    stage('Deploy Elastic Agent Pipeline') {
      steps {
        retry(2) {
          echo "Deploying ... demo-ea-with_secret"
          script {
            def fileContent = readFile file: '/tmp/pipeline_deployment/sample_pipeline-ea.conf'
            // Try to replace double quotes with single quotes
            fileContent = fileContent.replaceAll('"', "'")
            fileContent = fileContent.replaceAll('\r', '\\\\r')
            fileContent = fileContent.replaceAll('\n', '\\\\n')
            env.textData = fileContent
          }
          echo "${env.textData}"
          httpRequest httpMode: 'PUT', url: 'https://apm-dev-ac.es.us-east-2.aws.elastic-cloud.com/_logstash/pipeline/demo-ea-with_secret',
          acceptType: 'APPLICATION_JSON',
          contentType: 'APPLICATION_JSON',
          authentication: 'o11y_es_ingest',
          requestBody: """{
            "description": "", 
            "last_modified": "2025-06-04T02:50:51.250Z",
            "pipeline_metadata": {
              "type": "logstash_pipeline",
              "version": "1"
            },
            "username": "O11y-cicd-user",
            "pipeline": "${env.textData}",
            "pipeline_settings": {
              "pipeline.workers": 1,
              "pipeline.batch.size": 125,
              "pipeline.batch.delay": 50,
              "queue.type": "memory",
              "queue.max_bytes": "1gb",
              "queue.checkpoint.writes": 1024
            }
          }
          """
        }
      }
    }
  }
  post {
    always {
      sh 'rm -rf /tmp/pipeline_deployment/'
    }
  }
}
