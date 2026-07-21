pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ---------- Azure ----------
        AZURE_STORAGE_ACCOUNT = 'fqbuenostg'
        AZURE_FILE_SHARE      = 'webcontent'
        ACI_URL               = 'http://20.204.214.142'

        // ---------- AWS Production ----------
        SERVERS = 'ubuntu@10.0.10.99 ubuntu@10.0.11.115'
        DOCROOT = '/var/www/html'

        // ---------- Application ----------
        APP_SRC = '.'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "Building application..."

                    # Example builds:
                    # npm install
                    # npm test
                    # npm run build

                    echo "No build required."
                '''
            }
        }

        stage('Deploy to Staging (Azure ACI)') {
            steps {
                sh '''
                    echo "Uploading application to Azure File Share..."

                    az storage file upload-batch \
                        --account-name $AZURE_STORAGE_ACCOUNT \
                        --destination $AZURE_FILE_SHARE \
                        --source $APP_SRC \
                        --overwrite true \
                        --auth-mode login
                '''
            }
        }

        stage('Test Staging') {
            steps {
                sh '''
                    echo "Waiting for ACI..."
                    sleep 20

                    echo "Checking HTTP status..."

                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" $ACI_URL)

                    if [ "$STATUS" != "200" ]; then
                        echo "Expected HTTP 200 but got $STATUS"
                        exit 1
                    fi

                    echo "Checking page contents..."

                    curl -fs $ACI_URL | grep -q "Welcome to Cloud Demo"

                    echo "Staging validation passed."
                '''
            }
        }

        stage('Deploy to Production') {
            steps {

                sshagent(credentials: ['webserver-key']) {

                    sh '''
                        set -e

                        SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"

                        for HOST in $SERVERS
                        do
                            echo "Deploying to $HOST"

                            rsync -az --delete \
                                -e "ssh $SSH_OPTS" \
                                --rsync-path="sudo rsync" \
                                --exclude '.git' \
                                --exclude 'Jenkinsfile' \
                                "$APP_SRC/" \
                                "$HOST:$DOCROOT/"

                            ssh $SSH_OPTS $HOST \
                                "sudo systemctl reload apache2"

                            echo "$HOST deployment completed."
                        done
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Production deployment was skipped.'
        }

        always {
            cleanWs()
        }
    }
}
