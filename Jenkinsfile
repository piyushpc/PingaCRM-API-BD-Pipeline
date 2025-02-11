pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['Development', 'uat', 'prod'],
            description: 'Select the deployment environment: dev, uat, prod'
        )
    }

    environment {
        SVN_CREDENTIALS = 'svn-credentials-id'
        SSH_KEY_PATH = '/var/lib/jenkins/vkey.pem'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Selected Environment: ${params.ENVIRONMENT}"
                    
                    def envConfig = [
                        'Development': ['API_SERVER': 'ec2-13-234-54-22.ap-south-1.compute.amazonaws.com', 'SSH_CREDENTIALS': 'dev-ssh-credentials-id'],
                        'uat':        ['API_SERVER': 'apiuat.pingacrm.com', 'SSH_CREDENTIALS': 'uat-ssh-credentials-id'],
                        'prod':       ['API_SERVER': 'apiprod.pingacrm.com', 'SSH_CREDENTIALS': 'prod-ssh-credentials-id']
                    ]

                    if (!envConfig.containsKey(params.ENVIRONMENT)) {
                        error "Invalid environment: ${params.ENVIRONMENT}"
                    }

                    env.API_SERVER = envConfig[params.ENVIRONMENT]['API_SERVER']
                    env.SSH_CREDENTIALS = envConfig[params.ENVIRONMENT]['SSH_CREDENTIALS']

                    echo "Initializing deployment to ${params.ENVIRONMENT} on ${env.API_SERVER}"
                    echo "SSH Credentials: ${env.SSH_CREDENTIALS}"
                }
            }
        }

        stage('Stop Services') {
            steps {
                script {
                    echo "Stopping services on ${env.API_SERVER}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.API_SERVER} \
                        echo "[INFO] Stopping services..."
                        sudo systemctl stop aspnetcoreapp.service
                        sudo systemctl stop aspnetcorescheduler.service
                        sudo service apache2 stop
                    """
                }
            }
        }

        stage('Backup Old Code') {
            steps {
                script {
                    echo "Backing up old code on ${env.API_SERVER}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Backing up old code..."
                        cd /home/ubuntu
                        cp -R PingaCRM PingaCRM-\$(date +%Y%m%d%H%M%S)
EOF
                    """
                }
            }
        }

        stage('Checkout and Update Code') {
            steps {
                script {
                    echo "Checking out and updating code from SVN on ${env.API_SERVER}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    withCredentials([usernamePassword(credentialsId: env.SVN_CREDENTIALS, usernameVariable: 'SVN_USER', passwordVariable: 'SVN_PASS')]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                            echo "[INFO] Updating code from SVN..."
                            cd /home/ubuntu/PingaCRM
                            svn update --username $SVN_USER --password $SVN_PASS
EOF
                        """
                    }
                }
            }
        }

        stage('Update Configuration') {
            steps {
                script {
                    echo "Updating configuration for ${params.ENVIRONMENT}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Updating configuration for ${params.ENVIRONMENT}..."
                        cd /home/ubuntu
                        cp appsettings.${params.ENVIRONMENT}.json appsettings.${params.ENVIRONMENT}.json.\$(date +%Y%m%d%H%M%S).backup
                        cp appsettings.${params.ENVIRONMENT}.json PingaCRM/PingaCRM.API/appsettings.${params.ENVIRONMENT}.json
                        cp appsettings.${params.ENVIRONMENT}.json PingaCRM/PingaCRMScheduler/appsettings.${params.ENVIRONMENT}.json
EOF
                    """
                }
            }
        }

        stage('Build and Deploy') {
            steps {
                script {
                    echo "Building and deploying application on ${env.API_SERVER}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Building and deploying..."
                        cd /home/ubuntu/PingaCRM
                        sudo dotnet restore
                        sudo dotnet publish -o awscompiledcode
                        sudo chmod -R 777 /home/ubuntu/PingaCRM
                        sudo chown -R www-data:www-data /home/ubuntu/PingaCRM
EOF
                    """
                }
            }
        }

        stage('Start Services') {
            steps {
                script {
                    echo "Starting services on ${env.API_SERVER}"
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Starting services..."
                        sudo service apache2 start
                        sudo systemctl start aspnetcoreapp.service
                        sudo systemctl start aspnetcorescheduler.service
                        echo "[INFO] Checking logs..."
                        journalctl -u aspnetcoreapp.service | tail -n 100
                        journalctl -u aspnetcorescheduler.service | tail -n 100
EOF
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to ${params.ENVIRONMENT} completed successfully!"
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed. Check logs for details."
        }
    }
}
