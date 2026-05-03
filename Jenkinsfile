pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  parameters {
    string(
      name: 'IMAGE_TAG',
      defaultValue: 'latest',
      description: 'ruoyi 镜像 tag，例如 42-a3f8c1d。留空使用默认值 latest。'
    )
  }
  environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Verify Ansible') {
      steps {
        sh '''
          set -eu
          ansible --version
          ansible-playbook --version
        '''
      }
    }
    stage('Ansible Check') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'ANSIBLE_KEY', usernameVariable: 'ANSIBLE_USER'),
          file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS_FILE')
        ]) {
          sh '''
            set -e
            ANSIBLE_PRIVATE_KEY_FILE="$ANSIBLE_KEY" \
            ansible-playbook playbooks/site.yml \
              -i inventory/hosts.yml \
              -u "$ANSIBLE_USER" \
              --vault-password-file "$VAULT_PASS_FILE" \
              --extra-vars "ruoyi_image_tag=${IMAGE_TAG:latest}" \
              --check --diff
          '''
        }
      }
    }
    stage('Deploy') {
      when {
        anyOf {
          branch 'main'
          expression { env.GIT_BRANCH == 'origin/main' || env.BRANCH_NAME == 'main' }
        }
      }
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'ANSIBLE_KEY', usernameVariable: 'ANSIBLE_USER'),
          file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS_FILE')
        ]) {
          sh '''
            set -e
            ANSIBLE_PRIVATE_KEY_FILE="$ANSIBLE_KEY" \
            ansible-playbook playbooks/site.yml \
              -i inventory/hosts.yml \
              -u "$ANSIBLE_USER" \
              --vault-password-file "$VAULT_PASS_FILE" \
              --extra-vars "ruoyi_image_tag=${IMAGE_TAG:latest}"
          '''
        }
      }
    }
    stage('Smoke Test') {
      when {
        anyOf {
          branch 'main'
          expression { env.GIT_BRANCH == 'origin/main' || env.BRANCH_NAME == 'main' }
        }
      }
      steps {
        sh '''
          set -eu
          echo "等待 30s，留给 Spring Boot 完成初始化..."
          sleep 30
          result=$(curl -sf "http://100.117.7.75:9090/api/v1/query" \
            --data-urlencode 'query=min(probe_success{service="ruoyi"})' \
            | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data['data']['result']
if not results:
    print('0')
else:
    print(results[0]['value'][1])
")
          echo "probe_success = ${result}"
          if [ "${result}" != "1" ]; then
            echo "Smoke Test FAILED: probe_success=${result}"
            exit 1
          fi
          echo "Smoke Test PASSED"
        '''
      }
    }
  }
  post {
    success {
      withCredentials([string(credentialsId: 'feishu-webhook-url', variable: 'FEISHU_URL')]) {
        sh '''
          set -e
          curl -sf -X POST "$FEISHU_URL" \
            -H "Content-Type: application/json" \
            -d "{
              \"msg_type\": \"text\",
              \"content\": {
                \"text\": \"✅ [docker-infra] 部署完成：ruoyi:${IMAGE_TAG} 已上线，Smoke Test 通过。Job #${BUILD_NUMBER}\"
              }
            }"
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'feishu-webhook-url', variable: 'FEISHU_URL')]) {
        sh '''
          set -e
          curl -sf -X POST "$FEISHU_URL" \
            -H "Content-Type: application/json" \
            -d "{
              \"msg_type\": \"text\",
              \"content\": {
                \"text\": \"❌ [docker-infra] 部署失败：Job #${BUILD_NUMBER}，版本 ${IMAGE_TAG}，请查看 Jenkins。\"
              }
            }"
        '''
      }
    }
  }
}