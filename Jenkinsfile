pipeline {
  agent any
  environment {
        AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
  }
  stages {
    stage("Config validation.") {
      steps {
        echo "Testing ..."
        wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['AWS_ACCESS_KEY_ID'], var: 'SECRET'], [password: env['AWS_SECRET_ACCESS_KEY'], var: 'SECRET']]]) {
          sh '''
            /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
            '''
          sh '''
            echo "${AWS_SECRET_ACCESS_KEY}" | /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash add VAULT_SECRET
          '''
        }
        sh '''#!/bin/bash
        cp test.conf /tmp/test_jenkins.conf && cat /tmp/test_jenkins.conf
        if /usr/share/logstash/bin/logstash --path.settings /tmp/logstash -t -f /tmp/test_jenkins.conf | grep "Configuration OK"; then 
          echo "Syntax OK"
          exit 0
        else
          echo "Syntax Error"
          exit 1 
        fi
        '''
        sh '''
        /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
        '''
      }
    }
    stage('Update Local LS Keystore update') {
      steps {
          echo "Updating Keystore ..."
          wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['AWS_ACCESS_KEY_ID'], var: 'SECRET'], [password: env['AWS_SECRET_ACCESS_KEY'], var: 'SECRET']]]) {
            sh '''
              /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash remove VAULT_SECRET || true
              '''
            sh '''
              echo "${AWS_SECRET_ACCESS_KEY}" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add VAULT_SECRET
            '''
          }
      }
    }

    stage('Deploy') {
      steps {
        echo "Deploying ..."
        script {
          def fileContent = readFile file: '/tmp/test_jenkins.conf'
          /*fileContent = fileContent.replaceAll('\r\n', ' ')
          fileContent = fileContent.replaceAll('\n', ' ')*/
          fileContent = fileContent.replaceAll('\r', '\\r')
          fileContent = fileContent.replaceAll('\n', '\\n')
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
  post {
    always {
      sh 'rm /tmp/test_jenkins.conf'
    }
  }
}
