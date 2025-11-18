pipeline {
  agent any

  environment {
    SONAR_TOKEN = credentials('sonar')
    NEXUS_CRED  = credentials('nexus')
    DOCKER_HUB  = credentials('docker-hub')
    IMAGE_TAG   = "rakesh268/newmew:${env.BUILD_NUMBER ?: 'latest'}"
    NEXUS_UPLOAD_URL = "http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz"
    SONAR_HOST = "http://51.20.37.198:9000"
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'master', url: 'https://github.com/RakeshKasagani/newmew-javascript.git'
      }
    }

    stage('Debug environment') {
      steps {
        sh '''
          echo "USER: $(whoami) HOST: $(hostname)"
          echo "Workspace: $WORKSPACE"
          echo "PATH=$PATH"
          echo "which node:"
          which node || true
          node -v || true
          npm -v || true
          echo "which docker:"
          which docker || echo "docker not found"
        '''
      }
    }

    stage('Install & Build (auto-detect)') {
      steps {
        script {
          // prefer system node if available
          def nodeOnPath = sh(script: 'which node >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
          def dockerOnPath = sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'

          if (nodeOnPath) {
            echo "Using node from PATH on agent"
            sh '''
              node -v
              npm -v
              npm ci --no-audit --no-fund
              if npm run | grep -q " build"; then
                npm run build || echo "build failed but continuing"
              else
                echo "No build script found"
              fi
              ls -la
            '''
          } else if (dockerOnPath) {
            echo "System node not found; attempting to run build inside node:18-alpine Docker image"
            // use docker to run node commands with workspace mounted
            sh """
              docker run --rm -v "$WORKSPACE":"$WORKSPACE" -w "$WORKSPACE" node:18-alpine /bin/sh -c \\
                "apk add --no-cache python3 make g++ >/dev/null 2>&1 || true; \\
                 node -v; npm -v || true; \\
                 npm ci --no-audit --no-fund || { echo 'npm ci failed'; exit 2; }; \\
                 if npm run | grep -q ' build'; then npm run build || echo 'build failed but continuing'; fi; \\
                 ls -la"
            """
          } else {
            error """No node/npm available on agent PATH and docker is not usable.
To fix:
  - install Node (system-wide) on the agent OR
  - configure the Jenkins NodeJS plugin (Manage Jenkins → Global Tool Configuration) with a Node installation, or
  - enable Docker for the Jenkins user so the pipeline can use the node Docker image.
"""
          }
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          def sonarInstalled = sh(script: 'command -v sonar-scanner >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
          def dockerOnPath = sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'

          if (sonarInstalled) {
            echo "Running sonar-scanner from agent"
            sh """
              sonar-scanner \
                -Dsonar.projectKey=nodeapp \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST} \
                -Dsonar.login=${SONAR_TOKEN}
            """
          } else if (dockerOnPath) {
            echo "sonar-scanner not installed; trying sonar-scanner docker image"
            sh """
              docker run --rm -v "$WORKSPACE":"$WORKSPACE" -w "$WORKSPACE" \\
                -e SONAR_LOGIN=${SONAR_TOKEN} \\
                sonarsource/sonar-scanner-cli:latest \\
                -Dsonar.projectKey=nodeapp \\
                -Dsonar.sources=. \\
                -Dsonar.host.url=${SONAR_HOST} \\
                -Dsonar.login=\$SONAR_LOGIN
            """
          } else {
            echo "Skipping SonarQube analysis: no sonar-scanner installed and docker not usable."
          }
        }
      }
    }

    stage('Create Artifact & Upload to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "Creating TAR artifact..."
            tar --ignore-failed-read --warning=no-file-changed -czf newmew.tar.gz * || { echo "tar failed"; exit 1; }
            echo "Uploading to Nexus..."
            curl -v -u $NEXUS_USER:$NEXUS_PASS --upload-file newmew.tar.gz ${NEXUS_UPLOAD_URL} || { echo "Nexus upload failed"; exit 1; }
          '''
        }
      }
    }

    stage('Build Docker Image') {
      when {
        expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' }
      }
      steps {
        sh '''
          echo "Building Docker image ${IMAGE_TAG}"
          docker build -t ${IMAGE_TAG} .
          docker images | head -n 20
        '''
      }
    }

    stage('Push Docker Image') {
      when {
        expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_TAG} || { echo "docker push failed"; exit 1; }
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished!"
      cleanWs()
    }
    success {
      echo "Pipeline succeeded."
    }
    failure {
      echo "Pipeline failed — check the logs above."
    }
  }
}
