// docker-infra CD：拉代码 → Ansible 预检 →（仅 main）全量部署 → Prometheus 烟测 → 飞书通知。
// 镜像构建与推 Registry 在 ruoyi 仓库的独立 Jenkins Job；本文件只负责用 IMAGE_TAG 驱动 Ansible 下发。
pipeline {
  agent any

  options {
    timestamps()              // 日志带时间戳，便于对齐 Ansible / 业务日志
    disableConcurrentBuilds() // 避免同一 Job 并发改同一批节点
  }

  parameters {
    // 与 ruoyi CI 产出的 tag 对齐（如 BUILD-SHA）；默认 latest，与 group_vars 一致
    string(
      name: 'IMAGE_TAG',
      defaultValue: 'latest',
      description: 'ruoyi 镜像 tag，例如 42-a3f8c1d。留空使用默认值 latest。'
    )
  }

  environment {
    // 首次 SSH 到各节点时不因 known_hosts 阻塞（密钥仍由 ansible-ssh-key 提供）
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {
    // 拉取当前 Job 绑定的 SCM（通常为 docker-infra 的 main）
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // 确认 Jenkins agent 上 ansible-playbook 可用，避免 credentials 阶段才失败
    stage('Verify Ansible') {
      steps {
        sh '''
          set -eu
          ansible --version
          ansible-playbook --version
        '''
      }
    }

    // 任意分支都跑：dry-run，不写远端；尽早发现 playbook / vault / 模板问题
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
              --extra-vars "ruoyi_image_tag=${IMAGE_TAG:-latest}" \
              --check --diff
          '''
        }
      }
    }

    // 仅 main：真实执行 site.yml，将 ruoyi 镜像版本写入各 app 节点并 compose 滚动
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
              --extra-vars "ruoyi_image_tag=${IMAGE_TAG:-latest}"
          '''
        }
      }
    }

    // 仅 main：用 Prometheus 上 blackbox 的 probe_success 判定业务探测，避免 Jenkins 直连业务 IP
    // 查询地址需与当前监控主节点（如 gz-01）一致，变更节点时请同步架构快照与此处或改为注入变量
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
    // 整次 Pipeline 成功（含非 main 仅跑 Check 的成功）：发飞书；main 上通常表示部署+烟测通过
    success {
      withCredentials([string(credentialsId: 'feishu-webhook-url-jenkins-notify', variable: 'FEISHU_URL')]) {
        sh '''
          set -eu
          cat > feishu-payload.json <<EOF
{
  "msg_type": "text",
  "content": {
    "text": "✅ [docker-infra] 部署完成：ruoyi:${IMAGE_TAG:-latest} 已上线，Smoke Test 通过。Job #${BUILD_NUMBER}"
  }
}
EOF
          curl -sS -f -X POST "$FEISHU_URL" \
            -H "Content-Type: application/json" \
            --data-binary @feishu-payload.json
        '''
      }
    }
    // 任一 stage 失败：提醒人工看 Jenkins 控制台与 Ansible 输出
    failure {
      withCredentials([string(credentialsId: 'feishu-webhook-url-jenkins-notify', variable: 'FEISHU_URL')]) {
        sh '''
          set -eu
          cat > feishu-payload.json <<EOF
{
  "msg_type": "text",
  "content": {
    "text": "❌ [docker-infra] 部署失败：Job #${BUILD_NUMBER}，版本 ${IMAGE_TAG:-latest}，请查看 Jenkins。"
  }
}
EOF
          curl -sS -f -X POST "$FEISHU_URL" \
            -H "Content-Type: application/json" \
            --data-binary @feishu-payload.json
        '''
      }
    }
  }
}
