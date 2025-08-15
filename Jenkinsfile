pipeline {
    agent any

    tools {
        nodejs "nodejs"
    }

    environment {
        DEPLOY_SSH_KEY = credentials('AWS_INSTANCE_SSH')
        PRODUCTION_IP_ADDRESS = 'your.ec2.ip.here' // Replace with actual IP or set globally
    }

    stages {
        stage('Install Packages') {
            steps {
                sh 'yarn install'
            }
        }

        stage('Run the App') {
            steps {
                sh 'yarn start:pm2'
                sleep 10 // Increased to allow app startup
            }
        }

        stage('Test the App') {
            steps {
                script {
                    retry(3) {
                        sleep 5
                        sh 'curl --fail http://localhost:3000/health'
                    }
                }
            }
        }

        stage('PM2 Logs') {
            steps {
                sh 'pm2 logs todos-app --lines 50'
            }
        }

        stage('Stop the App') {
            steps {
                sh 'pm2 stop todos-app'
            }
        }

        stage('Add Host to known_hosts') {
            steps {
                sh '''
                    ssh-keyscan -H $PRODUCTION_IP_ADDRESS >> /var/lib/jenkins/.ssh/known_hosts
                '''
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'AWS_INSTANCE_SSH', keyFileVariable: 'KEY')]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $KEY ubuntu@$PRODUCTION_IP_ADDRESS '
                            if [ ! -d "todos-app" ]; then
                                git clone https://github.com/Veeras-code/todos-app.git todos-app
                                cd todos-app
                            else
                                cd todos-app
                                git pull
                            fi

                            yarn install

                            if pm2 describe todos-app > /dev/null ; then
                                pm2 restart todos-app
                            else
                                yarn start:pm2
                            fi
                        '
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
