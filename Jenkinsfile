pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
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
            set -eu
            ANSIBLE_PRIVATE_KEY_FILE="$ANSIBLE_KEY" \
            ansible-playbook playbooks/site.yml \
              -i inventory/hosts.yml \
              -u "$ANSIBLE_USER" \
              --vault-password-file "$VAULT_PASS_FILE" \
              --check --diff
          '''
        }
      }
    }
    stage('Deploy') {
      when {
        branch 'main'
      }
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'ANSIBLE_KEY', usernameVariable: 'ANSIBLE_USER'),
          file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS_FILE')
        ]) {
          sh '''
            set -eu
            ANSIBLE_PRIVATE_KEY_FILE="$ANSIBLE_KEY" \
            ansible-playbook playbooks/site.yml \
              -i inventory/hosts.yml \
              -u "$ANSIBLE_USER" \
              --vault-password-file "$VAULT_PASS_FILE"
          '''
        }
      }
    }
  }
}
