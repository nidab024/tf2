pipeline {
    agent any
    
    environment {
        // AWS region to deploy to
        AWS_DEFAULT_REGION = 'ap-south-1'
        TERRAFORM_VERSION = '1.5.0'
        // Working directory for Terraform files - should be '.' for root directory
        TF_WORKSPACE = '.'
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
                // Checkout the specific repository
                git branch: 'main', 
                    url: 'https://github.com/nidab024/tf2.git'
            }
        }
        
        stage('Test AWS Access') {
            steps {
                script {
                    echo 'Testing AWS credentials...'
                    sh '''
                        aws sts get-caller-identity
                        aws ec2 describe-regions --region $AWS_DEFAULT_REGION --output table
                        echo "AWS credentials are working correctly!"
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
        
        stage('Terraform Security Scan') {
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Running Terraform security scan...'
                        sh '''
                            # Install tfsec if not available
                            if ! command -v tfsec &> /dev/null; then
                                echo "Installing tfsec..."
                                curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
                                sudo mv tfsec /usr/local/bin/
                            fi
                            
                            # Run security scan
                            tfsec . --format json --out tfsec-results.json || true
                            tfsec . --format default || true
                        '''
                        
                        // Archive security scan results
                        archiveArtifacts artifacts: 'tfsec-results.json', allowEmptyArchive: true
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
                params.ACTION == 'apply'
            }
            steps {
                dir("${TF_WORKSPACE}") {
                    script {
                        echo 'Displaying Terraform outputs...'
                        sh '''
                            echo "=== Terraform State Summary ==="
                            terraform show -no-color
                            echo ""
                            echo "=== Terraform Outputs ==="
                            terraform output -no-color -json > terraform-outputs.json || echo "No outputs defined"
                            terraform output -no-color || echo "No outputs defined"
                        '''
                        
                        // Archive outputs
                        archiveArtifacts artifacts: 'terraform-outputs.json', allowEmptyArchive: true
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up workspace...'
            // Archive Terraform state files for backup
            archiveArtifacts artifacts: 'terraform.tfstate*', allowEmptyArchive: true
            // Clean workspace but preserve .terraform directory for faster subsequent runs
            sh 'find . -name "*.tfplan" -delete || true'
            sh 'find . -name "tfdestroy" -delete || true'
        }
        success {
            echo 'Pipeline completed successfully!'
            // Send success notification
            emailext (
                subject: "✅ Terraform Deployment Success - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Terraform ${params.ACTION} completed successfully for ${env.JOB_NAME} build #${env.BUILD_NUMBER}",
                to: "${env.CHANGE_AUTHOR_EMAIL ?: env.BUILD_USER_EMAIL ?: 'admin@company.com'}"
            )
        }
        failure {
            echo 'Pipeline failed!'
            // Send failure notification
            emailext (
                subject: "❌ Terraform Deployment Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Terraform ${params.ACTION} failed for ${env.JOB_NAME} build #${env.BUILD_NUMBER}. Please check the console output.",
                to: "${env.CHANGE_AUTHOR_EMAIL ?: env.BUILD_USER_EMAIL ?: 'admin@company.com'}"
            )
        }
    }
}
