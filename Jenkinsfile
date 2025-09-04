pipeline {
    agent any
    
    // Define environment variables for secrets
    environment {
        PROXMOX_API_URL = credentials('proxmox-api-url')
        PROXMOX_API_TOKEN_ID = credentials('proxmox-api-token-id')
        PROXMOX_API_TOKEN_SECRET = credentials('proxmox-api-token-secret')
        SSH_KEY_PATH = credentials('jenkins-ssh-key-path')
    }
    
    stages {
        stage('Checkout') {
            steps {
                // This automatically checks out your GitHub repository
                checkout scm
            }
        }
        
        stage('Terraform Init & Plan') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh '''
                        terraform plan \
                        -var="proxmox_api_url=$PROXMOX_API_URL" \
                        -var="proxmox_api_token_id=$PROXMOX_API_TOKEN_ID" \
                        -var="proxmox_api_token_secret=$PROXMOX_API_TOKEN_SECRET" \
                        -out=tfplan
                    '''
                }
            }
        }
        
        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform apply -auto-approve tfplan'
                    
                    // Save VM IP to environment variable for Ansible
                    script {
                        env.VM_IP = sh(
                            script: 'terraform output -raw vm_ip',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }
        
        stage('Wait for VM Boot') {
            steps {
                echo "Waiting for VM to boot. IP: ${env.VM_IP}"
                sleep(time: 2, unit: 'MINUTES')
            }
        }
        
        stage('Create Ansible Inventory') {
            steps {
                dir('ansible') {
                    sh '''
                        echo "[webservers]" > inventory
                        echo "webserver ansible_host=${VM_IP} ansible_user=root ansible_ssh_private_key_file=${SSH_KEY_PATH}" >> inventory
                        cat inventory
                    '''
                }
            }
        }
        
        stage('Run Ansible') {
            steps {
                dir('ansible') {
                    sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory playbook.yml -v'
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    sleep 10
                    curl -s --retry 5 --retry-delay 5 http://${VM_IP}/ | grep "Success"
                '''
                echo "Apache is running at http://${VM_IP}/"
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs for details."
        }
    }
}
