pipeline {
  agent any
  environment {
      SECRET1    = vault path: 'secrets', key: 'password1', vaultUrl: 'https://my-vault.com:8200', credentialsId: 'my-creds', engineVersion: "2"
      SECRET2    = vault path: 'secrets', key: 'password2', vaultUrl: 'https://my-vault.com:8200', credentialsId: 'my-creds', engineVersion: "2"
      NOT_SECRET = vault path: 'secrets', key: 'username', vaultUrl: 'https://my-vault.com:8200', credentialsId: 'my-creds', engineVersion: "2"
  }
  stages {
    stage("Config validation.") {
      steps {
        echo "Testing ..."
        sh '''#!/bin/bash
        cp test.conf /tmp/test_jenkins.conf && cat /tmp/test_jenkins.conf
        if /usr/share/logstash/bin/logstash -t -f /tmp/test_jenkins.conf | grep "Configuration OK"; then 
          echo "Syntax OK"
          exit 0
        else
          echo "Syntax Error"
          exit 1 
        fi
        '''
      }
    }
    stage('Update Local LS Keystore update') {
      steps {
        wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['SECRET1'], var: 'SECRET'], [password: env['SECRET2'], var: 'SECRET']]]) {
          sh '''
          echo "${SECRET1}" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add VAULT_SECRET
          '''
        }
      }
    }
    stage('Update remote LS Keystore update') {
      steps {
        wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env['SECRET1'], var: 'SECRET'], [password: env['SECRET2'], var: 'SECRET']]]) {
          sshagent(credentials: ['demo-ssh-id']) {
            sh '''
                [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                ssh-keyscan -t rsa,dsa logstash.domain >> ~/.ssh/known_hosts
                ssh -tt username@logstash.domain 'echo "${SECRET1}" | /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add VAULT_SECRET'
            '''
          }
        }
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
        httpRequest httpMode: 'PUT', url: 'https://observability.es.australiaeast.azure.elastic-cloud.com/_logstash/pipeline/demo-jenkins-with_secret',
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
