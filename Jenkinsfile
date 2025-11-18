pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar')
        NEXUS_CRED  = credentials('nexus')
        DOCKER_HUB  = credentials('docker-hub')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/RakeshKasagani/newmew-javascript.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Node version:"
                    node -v

                    echo "NPM version:"
                    npm -v

                    echo "Installing Node dependencies..."
                    npm install --no-audit --no-fund
                '''
            }
        }

        stage('sonar') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=nodeapp \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://51.20.37.198:9000/
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId:'nexus',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {

                    sh '''
                        echo "Creating TAR..."
                        tar --ignore-failed-read --warning=no-file-changed \
                            -czf newmew.tar.gz *

                        echo "Uploading TAR to Nexus..."
                        curl -v -u $NEXUS_USER:$NEXUS_PASS \
                            --upload-file newmew.tar.gz \
                            http://51.20.37.198:8081/repository/newmew.tar.gz
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t rakesh268/newmew:latest .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push kishangollamudi/nodeapp:latest
                    '''
                }
            }
        }
    }   // ✅ THIS was missing (closing stages)

    post {
        always {
            echo "Pipeline finished!"
        }
    }
}
