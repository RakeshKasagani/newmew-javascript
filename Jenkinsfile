pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar')
        NEXUS_CRED = credentials('nexus')
        DOCKER_HUB = credentials('docker-hub')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/RakeshKasagani/newmew-javascript.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Node version:"
                    node -v

                    echo "NPM version:"
                    npm -v

                    echo Installing Node dependencies...
                    npm install --no-audit --no-fund
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('My-Sonar') {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=newmew \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://13.51.204.28:9000/
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Upload to nexus') {
            steps {
                sh '''
                    curl -v -u $NEXUS_CRED_USR:$NEXUS_CRED_PSW \
                        --upload-file nodeapp.zip \
                        http://13.51.204.28:8081/repository/nodejs/newmew.zip
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t rakesh268/newmew:latest .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo $DOCKER_HUB_PSW | docker login -u $DOCKER_HUB_USR --password-stdin
                    docker push rakesh268/newmew:latest
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline finished!"
        }
    }
}
