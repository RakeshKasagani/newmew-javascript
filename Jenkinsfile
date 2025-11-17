    agent any

    tools {
        nodejs 'NodeJS-20'
    }

    environment {
        DOCKER_IMAGE = "rakesh268/newmew"
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Running npm install..."
                sh "npm install"
            }
        }

        stage('Run Tests') {
            steps {
                echo "Skipping tests..."
                sh 'echo "Tests skipped"'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."

                script {
                    // Load Sonar Scanner Tool
                    def scannerHome = tool 'SonarScanner'

                    withCredentials([string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('My-Sonar') {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                  -Dsonar.projectKey=newmew \
                                  -Dsonar.sources=. \
                                  -Dsonar.host.url=$SONAR_HOST_URL \
                                  -Dsonar.login=$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing image to DockerHub..."

                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh """
                        echo $DH_PASS | docker login -u $DH_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}
