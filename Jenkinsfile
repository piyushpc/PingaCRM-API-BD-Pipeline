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
        API_SERVER = ''
        SSH_CREDENTIALS = ''
        SVN_CREDENTIALS = 'svn-credentials-id'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Selected Environment: ${params.ENVIRONMENT}"
                    echo "Available Environments: dev, uat, prod"
        
                    def serverMap = [
                        'Development': ['ec2-13-234-54-22.ap-south-1.compute.amazonaws.com', 'dev-ssh-credentials-id'],
                        'uat': ['apiuat.pingacrm.com', 'uat-ssh-credentials-id'],
                        'prod': ['apiprod.pingacrm.com', 'prod-ssh-credentials-id']
                    ]

                    if (serverMap.containsKey(params.ENVIRONMENT)) {
                        env.API_SERVER = serverMap[params.ENVIRONMENT][0]
                        env.SSH_CREDENTIALS = serverMap[params.ENVIRONMENT][1]
                    } else {
                        error "Invalid environment: ${params.ENVIRONMENT}"
                    }

                    echo "Deploying to: ${env.API_SERVER}"
                    echo "Using SSH Credentials: ${env.SSH_CREDENTIALS}"
                }
            }
        }

        stage('Stop Services') {
            steps {
                script {
                    if (!env.API_SERVER || !env.SSH_CREDENTIALS) {
                        error "Environment variables not set properly!"
                    }
                }
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Stopping services..."
                        sudo systemctl stop aspnetcoreapp.service
                        sudo systemctl stop aspnetcorescheduler.service
                        sudo service apache2 stop
EOF
                    """
                }
            }
        }

        stage('Backup Old Code') {
            steps {
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
                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Updating configuration for ${params.ENVIRONMENT} environment..."
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
            echo "Deployment to ${params.ENVIRONMENT} environment completed successfully!"
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} environment failed. Check logs for details."
        }
    }
}
