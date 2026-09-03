pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME     = 'jenkins-node-demo'
        CONTAINER_NAME = 'jenkins-node-demo'
        APP_PORT       = '3000'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Node version:"
                    node -v

                    echo "NPM version:"
                    npm -v

                    echo "Installing dependencies..."
                    npm ci || npm install
                '''
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                        -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Stopping old container..."

                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    echo "Starting new container..."

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:3000 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Container started."

                    docker ps
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    echo "Waiting for application..."

                    for i in {1..10}; do

                        if curl -fsS http://localhost:${APP_PORT} | grep -q "Hello, World!"; then
                            echo "================================="
                            echo "Application is LIVE"
                            echo "Port: ${APP_PORT}"
                            echo "================================="
                            exit 0
                        fi

                        echo "Application not ready... retrying"
                        sleep 2
                    done

                    echo "================================="
                    echo "Application FAILED"
                    echo "================================="

                    docker logs ${CONTAINER_NAME}

                    exit 1
                '''
            }
        }
    }

    post {

        success {
            echo "================================="
            echo "BUILD SUCCESSFUL"
            echo "Build #${BUILD_NUMBER} deployed"
            echo "Application: http://<EC2-PUBLIC-IP>:3000"
            echo "================================="
        }

        failure {
            echo "================================="
            echo "BUILD FAILED"
            echo "Build #${BUILD_NUMBER} failed"
            echo "================================="
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}
