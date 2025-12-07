
pipeline {
  agent any

  parameters {
    string(name: 'AWS_REGION',        defaultValue: 'ap-south-1', description: 'AWS region')
    string(name: 'TF_WORKSPACE',      defaultValue: 'default',    description: 'Terraform workspace')
    string(name: 'EC2_INSTANCE_TYPE', defaultValue: 't3.micro',   description: 'EC2 instance type')
    string(name: 'EC2_KEY_NAME',      defaultValue: '',           description: 'Existing EC2 Key Pair name (optional)')
    booleanParam(name: 'AUTO_APPLY',  defaultValue: false,        description: 'Skip approval and auto-apply')
  }

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws_access_key_id')
    AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key')
  }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  stages {
    stage('Checkout') {
      steps {
        // If job is Pipeline from SCM, Jenkins auto injects SCM; 'checkout scm' works.
        checkout scm
      }
    }

    stage('Terraform Init') {
      steps {
        dir('terraform') {
          sh 'terraform init -input=false'
        }
      }
    }

    stage('Select/Create Workspace') {
      steps {
        dir('terraform') {
          sh """
            set -e
            terraform workspace list | grep -q "${TF_WORKSPACE}" || terraform workspace new "${TF_WORKSPACE}"
            terraform workspace select "${TF_WORKSPACE}"
          """
        }
      }
    }

    stage('Terraform Plan') {
      steps {
        dir('terraform') {
          sh """
            terraform plan \
              -var aws_region="${AWS_REGION}" \
              -var vpc_cidr="10.0.0.0/16" \
              -var public_subnet_cidr="10.0.1.0/24" \
              -var instance_type="${EC2_INSTANCE_TYPE}" \
              -var key_name="${EC2_KEY_NAME}" \
              -input=false -out=tfplan
          """
        }
      }
    }

    stage('Approval') {
      when {
        expression { return !params.AUTO_APPLY }
      }
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          input message: 'Apply Terraform plan?'
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        dir('terraform') {
          sh 'terraform apply -input=false -auto-approve tfplan'
        }
      }
    }

    stage('Outputs') {
      steps {
        dir('terraform') {
          sh 'terraform output -json > tf-outputs.json || true'
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

