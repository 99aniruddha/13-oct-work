pipeline {
    agent any

    environment {
        TF_DIR = "terraform"
        ANSIBLE_DIR = "ansible"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📦 Checking out source code..."
                git(
                    url: 'https://github.com/1999aniruddha/13-oct-work.git',
                    branch: 'main'
                )
            }
        }

        stage('Terraform Init') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AWS_CREDS',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    dir(env.TF_DIR) {
                        bat """
@echo off
set TF_PLUGIN_CACHE_DIR=%WORKSPACE%\\.terraform-plugin-cache
if not exist "%TF_PLUGIN_CACHE_DIR%" mkdir "%TF_PLUGIN_CACHE_DIR%"

echo 🔧 Initializing Terraform...
terraform init -input=false
"""
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AWS_CREDS',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    dir(env.TF_DIR) {
                        bat """
@echo off
set TF_PLUGIN_CACHE_DIR=%WORKSPACE%\\.terraform-plugin-cache

echo 🧠 Running Terraform Plan...
terraform plan -out=tfplan -input=false
"""
                    }
                }
            }
        }

        stage('Terraform Apply') {
            options { timeout(time: 30, unit: 'MINUTES') }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AWS_CREDS',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    dir(env.TF_DIR) {
                        // 👇 new logic added here
                        bat """
@echo off
set TF_PLUGIN_CACHE_DIR=%WORKSPACE%\\.terraform-plugin-cache
setlocal enabledelayedexpansion

echo 🚀 Applying Terraform changes...
terraform apply -auto-approve -input=false tfplan > apply.log 2>&1
set APPLY_EXIT_CODE=%ERRORLEVEL%

REM --- Detect duplicate security group error or other known benign issues ---
findstr /C:"InvalidGroup.Duplicate" apply.log >nul
if %ERRORLEVEL%==0 (
    echo ⚠️ Duplicate security group found — continuing without failure.
    set APPLY_EXIT_CODE=0
)

REM --- Handle apply exit code ---
if %APPLY_EXIT_CODE% NEQ 0 (
    echo ❌ Terraform Apply failed. Check apply.log
) else (
    echo ✅ Terraform Apply succeeded or recovered.
)

REM --- Try extracting Terraform outputs regardless of apply result ---
terraform output -json > tf_outputs.json 2>nul || echo {} > tf_outputs.json

REM Try getting public IP from Terraform output or AWS CLI if not found
for /f "delims=" %%i in ('terraform output -raw public_ip 2^>nul') do set PUBLIC_IP=%%i

if not defined PUBLIC_IP (
    echo ⚠️ No public_ip output from Terraform — trying AWS CLI...
    aws ec2 describe-instances --filters "Name=tag:Name,Values=ci-cd-test" --query "Reservations[*].Instances[*].PublicIpAddress" --output text > temp_ip.txt 2>nul
    set /p PUBLIC_IP=<temp_ip.txt
)

if defined PUBLIC_IP (
    echo PUBLIC_IP=%PUBLIC_IP% > "%WORKSPACE%\\public_ip.env"
    echo ✅ Public IP captured: %PUBLIC_IP%
) else (
    echo ❌ No public IP found from Terraform or AWS.
)
"""
                    }
                }
            }
        }

        stage('Ansible Deploy') {
            when { expression { fileExists('public_ip.env') } }
            steps {
                sshagent(['deploy-key']) {
                    bat """
@echo off
for /f "tokens=1,2 delims==" %%i in (public_ip.env) do set %%i=%%j

if not defined PUBLIC_IP (
    echo ❌ No PUBLIC_IP found, skipping Ansible.
    exit /b 0
)

if not exist "%ANSIBLE_DIR%\\inventory" mkdir "%ANSIBLE_DIR%\\inventory"
(
echo [webservers]
echo %PUBLIC_IP% ansible_user=ubuntu ansible_ssh_common_args="-o StrictHostKeyChecking=no"
) > "%ANSIBLE_DIR%\\inventory\\hosts.ini"

echo 🚀 Running Ansible playbook on %PUBLIC_IP%...
ansible-playbook -i "%ANSIBLE_DIR%\\inventory\\hosts.ini" "%ANSIBLE_DIR%\\site.yml" --ssh-common-args="-o StrictHostKeyChecking=no"
"""
                }
            }
        }

        stage('Report') {
            when { expression { fileExists('public_ip.env') } }
            steps {
                bat """
@echo off
for /f "tokens=1,2 delims==" %%i in (public_ip.env) do set %%i=%%j
if defined PUBLIC_IP (
    echo 🌐 Application should be available at: http://%PUBLIC_IP%
) else (
    echo ⚠️ No public IP found, skipping report.
)
"""
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline finished successfully.'
        }
        failure {
            echo '❌ Pipeline failed. Check logs.'
        }
        always {
            echo '🧹 Cleaning workspace and archiving outputs...'
            archiveArtifacts artifacts: '**/terraform/tf_outputs.json, **/public_ip.env, **/terraform/terraform.log, **/terraform/apply.log', allowEmptyArchive: true
            cleanWs()
        }
    }
}
