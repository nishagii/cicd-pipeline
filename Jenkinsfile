pipeline {
    agent any

    parameters {
        string(
            name: 'TARGET_SERVER',
            defaultValue: '107.20.32.1',
            description: 'Apache Server Public IP'
        )
    }

    environment {
        S3_BUCKET = 'nj-cicd-artifacts'
        SONAR_PROJECT_KEY = 'cicd-demo'
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
                script {
                    env.ARTIFACT_NAME = "website-${BUILD_NUMBER}.zip"
                }

                sh '''
                zip -r $ARTIFACT_NAME index.html
                '''
            }
        }

        stage('Upload Artifact To S3') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]
                ]) {
                    sh '''
                    aws s3 cp \
                    $ARTIFACT_NAME \
                    s3://$S3_BUCKET/$ARTIFACT_NAME
                    '''
                }
            }
        }

        stage('Deploy To Apache') {
            steps {
                sshagent(['apache-ssh']) {
                    sh """
            ssh -o StrictHostKeyChecking=no ubuntu@${params.TARGET_SERVER} '

            sudo mkdir -p /var/www/html

            sudo rm -rf /var/www/html/*

            aws s3 cp \
            s3://$S3_BUCKET/$ARTIFACT_NAME \
            /tmp/$ARTIFACT_NAME

            sudo unzip -o \
            /tmp/$ARTIFACT_NAME \
            -d /var/www/html

            sudo systemctl restart apache2

            '
            """
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
                        http://${params.TARGET_SERVER}
                        """,
                        returnStdout: true
                    ).trim()

                    echo "HTTP STATUS CODE = ${statusCode}"

                    if (statusCode != '200') {
                        error('Deployment Validation Failed')
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful'

            echo "Artifact: ${ARTIFACT_NAME}"

            echo "Target Server: ${params.TARGET_SERVER}"
        }

        failure {
            echo 'Deployment Failed'
        }
    }
}
