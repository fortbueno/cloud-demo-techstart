pipeline {

    agent any

    environment {

        // Azure
        RESOURCE_GROUP = "rg-azuser7670_mml.local-e10FL"
        FILE_SHARE = "webcontent"
        STORAGE_ACCOUNT = "fqbuenostg"

        ACI_URL = "fqbuenolabel.centralindia.azurecontainer.io"

        // Production EC2 servers
        WEB1 = "ubuntu@10.0.10.99"
        WEB2 = "ubuntu@10.0.11.115"

        REMOTE_DIR = "/var/www/html"

    }

    stages {

        stage('Checkout') {

            steps {

                checkout scm

            }
        }

        stage('Azure Login') {

            steps {

                withCredentials([
                    string(credentialsId: 'azure-storage-account', variable: 'STORAGE_ACCOUNT'),
                ]) {
                    sh '''
                    az storage file upload-batch \
                    --account-name $STORAGE_ACCOUNT \
                    --account-key $STORAGE_KEY \
                    --destination $FILE_SHARE \
                    --source .
                    '''
}
            }
        }

        stage('Deploy to Azure Staging') {

            steps {

                sh '''

                echo "Uploading files to Azure File Share..."

                az storage file upload-batch \
                --account-name $STORAGE_ACCOUNT \
                --destination $FILE_SHARE \
                --source . \
                --overwrite \
                --auth-mode login

                '''

            }
        }

        stage('Wait for ACI') {

            steps {

                sleep(time:20, unit:'SECONDS')

            }

        }

        stage('HTTP Availability Test') {

            steps {

                sh '''

                STATUS=$(curl -o /dev/null -s -w "%{http_code}" $ACI_URL)

                if [ "$STATUS" != "200" ]; then

                    echo "ACI not reachable"

                    exit 1

                fi

                '''

            }

        }

        stage('Content Assertion') {

            steps {

                sh '''

                PAGE=$(curl -s $ACI_URL)

                echo "$PAGE"

                '''

            }

        }

        stage('Deploy Production') {

            steps {

                sshagent(['ec2-ssh-key']) {

                    sh '''

                    rsync -avz --delete \
                        --exclude='.git' \
                        -e "ssh -o StrictHostKeyChecking=no" \
                        ./ ubuntu@$WEB1:$REMOTE_DIR

                    ssh -o StrictHostKeyChecking=no ubuntu@$WEB1 <<EOF

                        sudo chown -R www-data:www-data $REMOTE_DIR

                        sudo systemctl restart apache2

EOF

                    rsync -avz --delete \
                        --exclude='.git' \
                        -e "ssh -o StrictHostKeyChecking=no" \
                        ./ ubuntu@$WEB2:$REMOTE_DIR

                    ssh -o StrictHostKeyChecking=no ubuntu@$WEB2 <<EOF

                        sudo chown -R www-data:www-data $REMOTE_DIR

                        sudo systemctl restart apache2

EOF

                    '''

                }

            }

        }

    }

    post {

        success {

            echo "Deployment Successful"

        }

        failure {

            echo "Deployment Failed"

        }

    }

}
