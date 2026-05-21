// docker-infra CD：拉代码 → 路径过滤（纯 docs 改动早退 NOT_BUILT）→ Ansible 预检 →（仅 main）全量部署 → Prometheus 烟测 → 飞书通知。
// 镜像构建与推 Registry 在 ruoyi 仓库的独立 Jenkins Job；本文件只负责用 IMAGE_TAG 驱动 Ansible 下发。
// 路径过滤之所以在 Pipeline 层而不是 SCM 层做：Jenkins Git plugin 在 webhook + ls-remote 轻量 polling 模式下只比对 HEAD SHA、不拉 commit 列表，
// 所以 Job UI 的 Excluded Regions 静默失效（曾在 #66 验证踩坑，详见 Docs/narratives/v1.7.md §7）。
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

    // 纯文档变更（Docs/、.cursor/、CHANGELOG.md、README.md、.gitignore）跳过后续部署。
    // 比对基准：GIT_PREVIOUS_SUCCESSFUL_COMMIT，避免被中间失败构建带歪。
    // fail-safe 原则：拿不到基准或 diff 为空时一律放行，宁可多跑一次也不要错过真改动。
    stage('Path Filter') {
      steps {
        script {
          def prevSha = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          if (!prevSha?.trim()) {
            echo "GIT_PREVIOUS_SUCCESSFUL_COMMIT 未设置（首次构建或历史已清空），fail-safe 正常运行 Pipeline。"
            return
          }

          def diffOut = sh(
            script: "git diff --name-only ${prevSha} HEAD 2>/dev/null || true",
            returnStdout: true
          ).trim()

          if (!diffOut) {
            echo "git diff ${prevSha}..HEAD 输出为空（无变更或本地缺该 commit），fail-safe 正常运行 Pipeline。"
            return
          }

          def files = diffOut.split('\n').findAll { it }
          def docPatterns = [
            ~/^Docs\/.*/,
            ~/^\.cursor\/.*/,
            ~/^CHANGELOG\.md$/,
            ~/^README\.md$/,
            ~/^\.gitignore$/,
          ]
          def nonDocFiles = files.findAll { f -> !docPatterns.any { p -> f ==~ p } }

          if (nonDocFiles.isEmpty()) {
            echo "本次 push 含 ${files.size()} 个改动文件，全部命中 docs 白名单，跳过部署:"
            files.each { echo "  - ${it}" }
            currentBuild.result = 'NOT_BUILT'
            currentBuild.description = 'Skipped: docs-only change'
            env.SKIP_PIPELINE = 'true'
          } else {
            echo "本次 push 含 ${nonDocFiles.size()} 个非 docs 改动，运行 Pipeline:"
            nonDocFiles.each { echo "  - ${it}" }
          }
        }
      }
    }

    // 确认 Jenkins agent 上 ansible-playbook 可用，避免 credentials 阶段才失败
    stage('Verify Ansible') {
      when { expression { env.SKIP_PIPELINE != 'true' } }
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
      when { expression { env.SKIP_PIPELINE != 'true' } }
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
        allOf {
          expression { env.SKIP_PIPELINE != 'true' }
          anyOf {
            branch 'main'
            expression { env.GIT_BRANCH == 'origin/main' || env.BRANCH_NAME == 'main' }
          }
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
        allOf {
          expression { env.SKIP_PIPELINE != 'true' }
          anyOf {
            branch 'main'
            expression { env.GIT_BRANCH == 'origin/main' || env.BRANCH_NAME == 'main' }
          }
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
