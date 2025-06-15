pipeline {
  agent any

  environment {
    TF_IN_AUTOMATION = 'true'
  }

  stages {
    stage('Terraform Init & Apply') {
      environment {
        ARM_CLIENT_ID       = credentials('azure-client-id')
        ARM_CLIENT_SECRET   = credentials('azure-client-secret')
        ARM_TENANT_ID       = credentials('azure-tenant-id')
        ARM_SUBSCRIPTION_ID = credentials('azure-subscription-id')
      }
      steps {
        dir('terraform') {
          sh '''
            terraform init
            terraform apply -auto-approve \
              -var="client_id=$ARM_CLIENT_ID" \
              -var="client_secret=$ARM_CLIENT_SECRET" \
              -var="tenant_id=$ARM_TENANT_ID" \
              -var="subscription_id=$ARM_SUBSCRIPTION_ID"
          '''
        }
      }
    }

    stage('Get Public IP from Terraform Output') {
      steps {
        script {
          env.VM_PUBLIC_IP = sh(
            script: 'terraform -chdir=terraform output -raw vm_ip',
            returnStdout: true
          ).trim()
          echo "Azure VM IP: ${env.VM_PUBLIC_IP}"
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible_ssh_key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            # Create dynamic inventory
            export ANSIBLE_HOST_KEY_CHECKING=False
            echo "[web]" > inventory.ini
            echo "$VM_PUBLIC_IP ansible_user=devops ansible_ssh_private_key_file=$SSH_KEY" >> inventory.ini

            # Run Ansible playbook
            ansible-playbook ansible/install_web.yml -i inventory.ini

            # Clean up
            rm -f inventory.ini
          '''
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        sh 'curl -s http://$VM_PUBLIC_IP | grep -i "devops project completed"'
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline executed successfully!"
    }
    failure {
      echo "❌ Pipeline failed. Check logs and validate stages."
    }
  }
}
