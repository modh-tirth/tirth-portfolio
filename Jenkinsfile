pipeline {
    agent any

    environment {
        IMAGE_NAME       = "portfolio"
        IMAGE_TAG        = "${BUILD_NUMBER}"
        SONAR_PROJECT    = "portfolio"
        S3_BUCKET        = "specialhub.shop"
        CF_DISTRIBUTION  = "EUD7SF0N29H3Z"
        AWS_REGION       = "ap-south-1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
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

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --no-progress \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-credentials']]) {
                    sh """
                        aws s3 sync . s3://${S3_BUCKET}/ \
                          --exclude '*' \
                          --include 'index.html' \
                          --region ${AWS_REGION} \
                          --delete
                    """
                }
            }
        }

        stage('CloudFront Invalidation') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-credentials']]) {
                    sh """
                        aws cloudfront create-invalidation \
                          --distribution-id ${CF_DISTRIBUTION} \
                          --paths '/*'
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
        success {
            echo "Pipeline succeeded — portfolio is live!"
        }
        failure {
            echo "Pipeline failed — check logs above."
        }
    }
}
