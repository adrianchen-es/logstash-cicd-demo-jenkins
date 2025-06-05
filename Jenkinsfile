pipeline {
  agent any
  stages {
    stage("Config validation.") {
      steps {
        echo "Testing ..."
        sh '''
          /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash remove VAULT_SECRET || true
          '''
        sh '''
          echo "SECRET1" | /usr/share/logstash/bin/logstash-keystore --path.settings /tmp/logstash add VAULT_SECRET || true
        '''
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
    stage('Clean up Local LS Keystore update') {
      steps {
          sh '''
          /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash remove VAULT_SECRET || true
          '''
      }
    }
    stage('Update Local LS Keystore update') {
      steps {
          sh '''
          echo "SECRET1" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add VAULT_SECRET || true
          '''
      }
    }

    stage('Deploy') {
      steps {
        script {
          fileContent = readFile file: '/tmp/test_jenkins.conf'
          fileContent = fileContent.replaceAll('\r\n', ' ')
          fileContent = fileContent.replaceAll('\n', ' ')
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
