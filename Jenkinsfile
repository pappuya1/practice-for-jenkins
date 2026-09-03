 pipeline {
    agent any

    triggers {
        githubPush()                 // auto-run on every push to main
    }

    options {
        timestamps()
        disableConcurrentBuilds()    // don't deploy two builds at once
    }

    environment {
        IMAGE_NAME     = 'jenkins-node-demo'
        CONTAINER_NAME = 'jenkins-node-demo'
        APP_PORT       = '3000'      // host port the app is exposed on
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install') {
            steps {
                sh 'node -v && npm -v'
                sh 'npm ci || npm install'
            }
        }

        stage('Test') {
            steps { sh 'npm test' }
        }

        // ---------- CI ends, CD begins ----------

        stage('Build image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .'
            }
        }

        stage('Deploy') {
            steps {
                // Recreate strategy: drop the old container, run the new image
                sh '''
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:3000 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Smoke test') {
            steps {
                // Wait for boot, then confirm the app actually responds
                sh '''
                    sleep 3
                    curl -fsS http://localhost:${APP_PORT} | grep -q "Hello, World!"
                    echo "App is live on port ${APP_PORT}"
                '''
            }
        }
    }

    post {
        success { echo "Deployed build #${BUILD_NUMBER} -> http://<EC2-PUBLIC-IP>:${APP_PORT}" }
        failure { echo "Build or deploy #${BUILD_NUMBER} failed" }
        always  { sh 'docker image prune -f || true' }   // reclaim disk from old images
    }
}
