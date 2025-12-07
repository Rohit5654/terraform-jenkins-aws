
pipeline {
  agent any

  parameters {
    string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region')
    string(name: 'TF_WORKSPACE', defaultValue: 'default', description: 'Terraform workspace')
    booleanParam(name: 'AUTO_APPLY', defaultValue: false, description: 'Auto-apply without manual approval')
  }

  environment {
    // If you set TF_WORKSPACE here, use Approach A above (no CLI select)
    TF_WORKSPACE = "${params.TF_WORKSPACE}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Terraform Init') {
      steps {
        dir('terraform') {
          sh '''
            set -euo pipefail
            terraform -version
            terraform init -input=false
          '''
        }
      }
    }

    stage('Select/Create Workspace') {
      steps {
        dir('terraform') {
          // Use ONE of the approaches:
          // --- Approach A (preferred if TF_WORKSPACE is set) ---
          sh '''
            set -euo pipefail
            echo "TF_WORKSPACE param value: '${TF_WORKSPACE}'"

            if ! terraform workspace list | sed 's/*//g' | awk '{$1=$1};1' | grep -qx "${TF_WORKSPACE}"; then
              terraform workspace new "${TF_WORKSPACE}"
            fi
            echo "Workspace pinned via TF_WORKSPACE. Proceeding..."
          '''
          // --- Approach B (requires unset TF_WORKSPACE) ---
          // sh '''
          //   set -euo pipefail
          //   unset TF_WORKSPACE
          //   terraform workspace list | sed 's/*//g' | awk '{$1=$1};1' | grep -qx "${params.TF_WORKSPACE}" || terraform workspace new "${params.TF_WORKSPACE}"
          //   terraform workspace select "${params.TF_WORKSPACE}"
          // '''
        }
      }
    }

    stage('Terraform Plan') {
      steps {
        dir('terraform') {
          sh '''
            set -euo pipefail
            terraform plan \
              -var aws_region="${AWS_REGION}" \
              -input=false -out=tfplan

            echo "Plan generated: tfplan"
          '''
        }
      }
    }

    stage('Approval') {
      when { expression { return !params.AUTO_APPLY } }
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          input message: 'Apply Terraform plan?'
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        dir('terraform') {
          sh '''
            set -euo pipefail
            terraform apply -input=false -auto-approve tfplan
          '''
        }
      }
    }

    stage('Outputs') {
      steps {
        dir('terraform') {
          sh '''
            set -euo pipefail
            terraform output -json > tf-outputs.json || true
          '''
          archiveArtifacts artifacts: 'tf-outputs.json', fingerprint: true
        }
      }
    }
  }

  post {
    success { echo 'Provisioning completed successfully.' }
    failure { echo 'Provisioning failed. Check logs.' }
  }
}

