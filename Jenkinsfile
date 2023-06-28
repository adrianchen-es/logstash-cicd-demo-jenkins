pipeline {
  agent any
  stages {
    stage("test") {
      steps {
        echo "Testing ..."
        sh '''#!/bin/bash
        cp test.conf /tmp/test_jenkins.conf && cat /tmp/test_jenkins.conf
        if sudo /usr/share/logstash/bin/logstash -t -f /tmp/test_jenkins.conf | grep "Configuration OK"; then 
          echo "Syntax OK"
          exit 0
        else
          echo "Syntax Error"
          exit 1 
        fi
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
        httpRequest httpMode: 'PUT', url: 'https://observability.es.australiaeast.azure.elastic-cloud.com/_logstash/pipeline/test_jenkins',
        acceptType: 'APPLICATION_JSON',
        contentType: 'APPLICATION_JSON',
        authentication: 'o11y_es_ingest',
        requestBody: """{
          "description": "", 
          "last_modified": "2021-01-02T02:50:51.250Z",
          "pipeline_metadata": {
            "type": "logstash_pipeline",
            "version": "1"
          },
          "username": "super-ingest",
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
