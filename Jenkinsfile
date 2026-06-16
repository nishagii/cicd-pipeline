pipeline {

    agent any

    environment {
        S3_BUCKET = "nj-cicd-artifacts"
        APACHE_SERVER = "107.20.32.1"
        SONAR_PROJECT_KEY = "cicd-demo"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {

                script {

                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {

                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.token=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Create Artifact') {
            steps {
                sh '''
                zip -r website.zip index.html
                '''
            }
        }

        stage('Upload Artifact To S3') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds']
                ]) {

                    sh '''
                    aws s3 cp website.zip s3://$S3_BUCKET/website.zip
                    '''
                }
            }
        }

        stage('Deploy To Apache') {

            steps {

                sshagent(['apache-ssh']) {

                    sh '''
                    scp -o StrictHostKeyChecking=no \
                    website.zip \
                    ubuntu@$APACHE_SERVER:/tmp/

                    ssh -o StrictHostKeyChecking=no \
                    ubuntu@$APACHE_SERVER '
                    sudo mkdir -p /var/www/html

                    sudo rm -rf /var/www/html/*

                    sudo unzip -o /tmp/website.zip \
                    -d /var/www/html

                    sudo systemctl restart apache2
                    '
                    '''
                }
            }
        }

        stage('Validate Deployment') {

            steps {

                script {

                    def statusCode = sh(
                        script: """
                        curl -o /dev/null \
                        -s \
                        -w "%{http_code}" \
                        http://${APACHE_SERVER}
                        """,
                        returnStdout: true
                    ).trim()

                    echo "HTTP STATUS CODE = ${statusCode}"

                    if (statusCode != "200") {
                        error("Deployment Validation Failed")
                    }
                }
            }
        }
    }

    post {

        success {
            echo 'Deployment Successful'
        }

        failure {
            echo 'Deployment Failed'
        }
    }
}