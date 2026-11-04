
pipeline {
    agent any

    environment {
        PYTHONUNBUFFERED = '1'
        REGISTRY_NAME = 'python-crud-api'
        CONTAINER_NAME = 'production-api-container'
        PORT_MAPPING = '5000:5000'
        VOLUME_NAME = 'production-api-data'
        DOCKER_NETWORK = 'jenkins-net'
    }

    stages {

        stage('1. Checkout Code') {
            steps {
                echo 'Checking out source code from GitHub'
                checkout scm
            }
        }

        stage('2. Get Git Commit SHA') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Docker Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('3. Code Syntax Verification') {
            steps {
                echo 'Checking Python syntax...'
                sh 'python3 -m py_compile app.py'
            }
        }

        stage('4. Build Docker Image') {
            steps {
                echo 'Building Docker image...'

                sh '''
                    docker build -t ${REGISTRY_NAME}:${IMAGE_TAG} .
                    docker tag ${REGISTRY_NAME}:${IMAGE_TAG} ${REGISTRY_NAME}:latest
                '''
            }
        }

        stage('5. Automated Deployment') {
            steps {
                echo 'Deploying application...'

                sh '''
                    docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    docker volume create ${VOLUME_NAME}

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --network ${DOCKER_NETWORK} \
                        -p ${PORT_MAPPING} \
                        -v ${VOLUME_NAME}:/var/lib/backend_data \
                        ${REGISTRY_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('6. Health Check') {
            steps {
                echo 'Checking API health...'

                sh '''
                    echo "Starting API Health Check..."

                    for i in $(seq 1 15); do

                        if curl -fsS http://${CONTAINER_NAME}:5000/health; then
                            echo ""
                            echo "Health Check SUCCESS"

                            docker logs --tail 20 ${CONTAINER_NAME}

                            exit 0
                        fi

                        echo "Waiting for API... attempt $i/15"
                        sleep 2
                    done

                    echo "Health Check FAILED"

                    docker logs --tail 50 ${CONTAINER_NAME}

                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Please check Console Output.'
        }
    }
}
