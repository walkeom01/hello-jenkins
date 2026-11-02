pipeline {
    agent any

    environment {
        AWS_REGION      = 'ap-south-1'
        AWS_ACCOUNT_ID  = '603694479177'
        ECR_REPOSITORY  = 'hello-jenkins'
        ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_NAME      = 'hello-jenkins'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        LOCAL_IMAGE     = "${IMAGE_NAME}:${BUILD_NUMBER}"
        ECR_IMAGE       = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build -t "$LOCAL_IMAGE" .
                '''
            }
        }

        stage('Test Nginx Configuration') {
            steps {
                sh '''
                    set -eu
                    docker run --rm "$LOCAL_IMAGE" nginx -t
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                    set -eu
                    docker tag "$LOCAL_IMAGE" "$ECR_IMAGE"
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-secret']
                ]) {
                    sh '''
                        set -eu

                        docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            -e AWS_SESSION_TOKEN \
                            -e AWS_DEFAULT_REGION="$AWS_REGION" \
                            amazon/aws-cli \
                            ecr get-login-password --region "$AWS_REGION" \
                        | docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"
                    '''
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                sh '''
                    set -eu
                    docker push "$ECR_IMAGE"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed. Image pushed: ${ECR_IMAGE}"
        }

        failure {
            echo 'Pipeline failed. Check the stage logs above for the error.'
        }

        always {
            sh '''
                docker image rm -f "$LOCAL_IMAGE" "$ECR_IMAGE" 2>/dev/null || true
            '''
        }
    }
}
