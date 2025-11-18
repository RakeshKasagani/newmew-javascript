pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar')
        NEXUS_CRED  = credentials('nexus')           // used only if you want env style access
        DOCKER_HUB  = credentials('docker-hub')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/RakeshKasagani/newmew-javascript.git'
            }
        }

        stage('Debug environment') {
            steps {
                sh '''
                    echo "Running on $(whoami) @ $(hostname)"
                    echo "Workspace: $WORKSPACE"
                    echo "PATH=$PATH"
                    which node || true
                    node -v 2>/dev/null || echo "node not on PATH"
                '''
            }
        }

        // Use the official Node image to guarantee node/npm are available
        stage('Install & Build (inside node container)') {
            steps {
                script {
                    docker.image('node:20').inside {
                        sh '''
                            echo "Node version:"
                            node -v

                            echo "NPM version:"
                            npm -v

                            echo "Installing Node dependencies (npm ci)..."
                            npm ci --no-audit --no-fund

                            echo "Run build (if defined)..."
                            if [ -f package.json ]; then
                              # prefer a build script if present
                              if npm run | grep -q " build"; then
                                npm run build || true
                              fi
                            fi
                        '''
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // run sonar-scanner in a container so the agent doesn't need it installed
                withSonarQubeEnv('My-Sonar') {
                    script {
                        docker.image('sonarsource/sonar-scanner-cli:latest').inside('-u root') {
                            sh """
                                sonar-scanner \
                                  -Dsonar.projectKey=nodeapp \
                                  -Dsonar.sources=. \
                                  -Dsonar.host.url=http://51.20.37.198:9000 \
                                  -Dsonar.login=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        stage('Create Artifact & Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        echo "Listing files to be archived..."
                        ls -la

                        echo "Creating TAR..."
                        tar --ignore-failed-read --warning=no-file-changed -czf newmew.tar.gz *

                        echo "Uploading TAR to nexus..."
                        curl -v -u $NEXUS_USER:$NEXUS_PASS \
                            --upload-file newmew.tar.gz \
                            http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image on agent..."
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
                        echo "Logging into Docker Hub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "Pushing image..."
                        docker push rakesh268/newmew:latest
                    '''
                }
            }
        }
    }   // end stages

    post {
        always {
            echo "Pipeline finished!"
            // optional: clean workspace to free space on agent
            cleanWs()
        }
    }
}
