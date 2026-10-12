pipeline {
    agent any

    environment {
        IMAGE_NAME = 'dydcjsrjaror/jenkins-cicd-demo'
        IMAGE_TAG  = "${BUILD_NUMBER}"
        CONTAINER_NAME = 'jenkins-cicd-demo-test'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checkout completed.'
                sh 'pwd'
                sh 'ls -la'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Verify Image') {
            steps {
                sh 'docker images ${IMAGE_NAME}'
            }
        }

        stage('Deploy to Test Server') {
            steps {
                sh '''
                    set -eu

                    docker pull ${IMAGE_NAME}:latest

                    OLD_IMAGE=$(docker inspect \
                      --format='{{.Image}}' ${CONTAINER_NAME})

                    docker rm -f ${CONTAINER_NAME}

                    if ! docker run -d \
                      --name ${CONTAINER_NAME} \
                      --restart unless-stopped \
                      -p 8082:80 \
                      ${IMAGE_NAME}:latest; then

                        echo "New deployment failed. Rolling back."
                        docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                        docker run -d \
                          --name ${CONTAINER_NAME} \
                          --restart unless-stopped \
                          -p 8082:80 \
                          "$OLD_IMAGE"
                        exit 1
                    fi

                    DEPLOY_OK=false

                    for i in $(seq 1 10); do
                        if curl -fsS http://127.0.0.1:8082/ >/dev/null; then
                            DEPLOY_OK=true
                            break
                        fi
                        sleep 3
                    done

                    if [ "$DEPLOY_OK" != "true" ]; then
                        echo "Health check failed. Rolling back."
                        docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                        docker run -d \
                          --name ${CONTAINER_NAME} \
                          --restart unless-stopped \
                          -p 8082:80 \
                          "$OLD_IMAGE"
                        exit 1
                    fi

                    echo "Deployment and HTTP health check succeeded."
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline and test deployment completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
        always {
            sh 'docker logout >/dev/null 2>&1 || true'
        }
    }
}
