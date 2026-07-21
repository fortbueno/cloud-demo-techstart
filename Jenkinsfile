pipeline {
    agent any

    environment {

        // Azure
        AZURE_VM = "fqbueno@4.224.63.150"
        AZURE_SHARE = "/mnt/aci-share"

        // AWS Production Servers
        WEB1 = "ubuntu@10.0.10.99"
        WEB2 = "ubuntu@10.0.11.115"

        // ACI Endpoint
        STAGING_URL = "fqbuenolabel.centralindia.azurecontainer.io"

        // Content expected from homepage
        EXPECTED_TEXT = "Milestone Project"

        // Jenkins SSH Credential ID
        SSH_CREDENTIAL = "az-key"
        AWS_CREDENTIAL = "webserver-key"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/fortbueno/cloud-demo-techstart.git'
            }
        }

        stage('Build') {
            steps {
                echo "Preparing application..."

                sh '''
                mkdir -p app_src

                rsync -av \
                    --exclude=.git \
                    --exclude=.github \
                    --exclude=Jenkinsfile \
                    ./ app_src/
                '''
            }
        }

        stage('Deploy to Azure Staging') {
            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh """

                    echo "Deploying to Azure VM..."

                    rsync -avz \
                        -e "ssh -o StrictHostKeyChecking=no" \
                        app_src/ \
                        ${AZURE_VM}:${AZURE_SHARE}/

                    """

                }
            }
        }

        stage('Staging Validation') {

            steps {

                script {

                    retry(10) {

                        sleep(time: 10, unit: 'SECONDS')

                        sh """

                        echo "Checking HTTP..."

                        curl --fail ${STAGING_URL}

                        """

                    }

                    sh """

                    echo "Checking page contents..."

                    curl -s ${STAGING_URL}

                    """

                }

            }

        }

        stage('Deploy Production') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh """

                    echo "Deploying Web Server 1..."

                    rsync -avz \
                      -e "ssh -o StrictHostKeyChecking=no" \
                      app_src/ \
                      ${WEB1}:/var/www/html/

                    ssh -o StrictHostKeyChecking=no ${WEB1} \
                      "sudo systemctl restart httpd"

                    """

                    sh """

                    echo "Deploying Web Server 2..."

                    rsync -avz \
                      -e "ssh -o StrictHostKeyChecking=no" \
                      app_src/ \
                      ${WEB2}:/var/www/html/

                    ssh -o StrictHostKeyChecking=no ${WEB2} \
                      "sudo systemctl restart httpd"

                    """

                }

            }

        }

    }

    post {

        success {
            echo "Deployment Successful!"
        }

        failure {
            echo "Pipeline Failed."
            echo "Production deployment was NOT executed."
        }

    }
}
