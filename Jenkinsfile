pipeline {
    agent any
    
    environment {
       
        TERRAFORM_VERSION = '1.5.0'
        // Working directory for Terraform files
        TF_WORKSPACE = 'tf2'
    }
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Select Terraform action to perform'
        )
        booleanParam(
            name: 'AUTO_APPROVE',
            defaultValue: false,
            description: 'Auto approve terraform apply/destroy (use with caution)'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Test AWS Access') {
            steps {
                script {
                    echo 'Testing AWS credentials...'
                    sh '''
                        aws sts get-caller-identity
                        aws ec2 describe-regions --region ap-south-1
                    '''
                }
            }
        }
        
        stage('Setup Terraform') {
            steps {
                script {
                    echo 'Setting up Terraform...'
                    sh '''
                        # Download and install Terraform if not already available
                        if ! command -v terraform &> /dev/null; then
                            wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                            unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                            sudo mv terraform /usr/local/bin/
                            rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                        fi
                        terraform version
                    '''
                }
            }
        }
        
        stage('Terraform Init') {
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Initializing Terraform...'
                        sh '''
                            terraform init -no-color
                        '''
                    }
                }
            }
        }
        
        stage('Terraform Validate') {
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Validating Terraform configuration...'
                        sh '''
                            terraform validate -no-color
                        '''
                    }
                }
            }
        }
        
        stage('Terraform Format Check') {
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Checking Terraform formatting...'
                        sh '''
                            terraform fmt -check=true -diff=true -no-color
                        '''
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            when {
                anyOf {
                    params.ACTION == 'plan'
                    params.ACTION == 'apply'
                }
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Creating Terraform execution plan...'
                        sh '''
                            terraform plan -no-color -out=tfplan
                        '''
                        
                        // Archive the plan file
                        archiveArtifacts artifacts: 'tfplan', fingerprint: true
                    }
                }
            }
        }
        
        stage('Terraform Apply') {
            when {
                params.ACTION == 'apply'
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        if (params.AUTO_APPROVE) {
                            echo 'Applying Terraform plan (auto-approved)...'
                            sh '''
                                terraform apply -no-color -auto-approve tfplan
                            '''
                        } else {
                            echo 'Applying Terraform plan...'
                            input message: 'Do you want to apply this Terraform plan?', ok: 'Apply'
                            sh '''
                                terraform apply -no-color -auto-approve tfplan
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Terraform Destroy Plan') {
            when {
                params.ACTION == 'destroy'
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Creating Terraform destroy plan...'
                        sh '''
                            terraform plan -destroy -no-color -out=tfdestroy
                        '''
                        
                        // Archive the destroy plan file
                        archiveArtifacts artifacts: 'tfdestroy', fingerprint: true
                    }
                }
            }
        }
        
        stage('Terraform Destroy') {
            when {
                params.ACTION == 'destroy'
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        if (params.AUTO_APPROVE) {
                            echo 'Destroying Terraform resources (auto-approved)...'
                            sh '''
                                terraform apply -no-color -auto-approve tfdestroy
                            '''
                        } else {
                            echo 'Destroying Terraform resources...'
                            input message: 'Are you sure you want to destroy all resources?', ok: 'Destroy'
                            sh '''
                                terraform apply -no-color -auto-approve tfdestroy
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Terraform Output') {
            when {
                anyOf {
                    params.ACTION == 'apply'
                }
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Displaying Terraform outputs...'
                        sh '''
                            terraform output -no-color
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
            // You can add notifications here (Slack, email, etc.)
        }
        failure {
            echo 'Pipeline failed!'
            // You can add failure notifications here
        }
    }
}
