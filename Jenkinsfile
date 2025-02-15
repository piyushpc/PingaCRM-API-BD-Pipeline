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
        API_SERVER = 'ec2-13-234-54-22.ap-south-1.compute.amazonaws.com'
        SSH_CREDENTIALS = 'dev-ssh-credentials-id'
        SVN_CREDENTIALS = 'svn-credentials-id'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Selected Environment: ${params.ENVIRONMENT}"
                    
                    switch (params.ENVIRONMENT) {
                        case 'Development':
                            env.API_SERVER = 'ec2-13-234-54-22.ap-south-1.compute.amazonaws.com'
                            env.SSH_CREDENTIALS = 'dev-ssh-credentials-id'
                            break
                        case 'uat':
                            env.API_SERVER = 'apiuat.pingacrm.com'
                            env.SSH_CREDENTIALS = 'uat-ssh-credentials-id'
                            break
                        case 'prod':
                            env.API_SERVER = 'apiprod.pingacrm.com'
                            env.SSH_CREDENTIALS = 'prod-ssh-credentials-id'
                            break
                        default:
                            error "Invalid environment: ${params.ENVIRONMENT}"
                    }
                    echo "Deploying to: ${env.API_SERVER}"
                }
            }
        }

        stage('Stop Services') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    chmod 600 ${SSH_KEY}
                    ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
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
                withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
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
                withCredentials([
                    sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY'),
                    usernamePassword(credentialsId: env.SVN_CREDENTIALS, usernameVariable: 'SVN_USER', passwordVariable: 'SVN_PASS')
                ]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Updating code from SVN..."
                        cd /home/ubuntu/PingaCRM
                        svn update --username $SVN_USER --password $SVN_PASS
EOF
                    """
                }
            }
        }

        stage('Update Configuration') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
                        echo "[INFO] Updating configuration..."
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
                withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
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
                withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@${env.API_SERVER} <<EOF
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
