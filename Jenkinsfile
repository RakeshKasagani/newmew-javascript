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

  // Make sure you've configured a NodeJS installation named "NodeJS_18"
  tools {
    nodejs 'NodeJS_18'
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
          echo "Which node/npm/docker:"
          which node || true
          node -v || true
          npm -v || true
          which docker || echo "docker not found"
        '''
      }
    }

    stage('Install & Build') {
      steps {
        sh '''
          echo "Node:"
          node -v
          npm -v

          echo "Installing dependencies (npm ci)..."
          npm ci --no-audit --no-fund

          if npm run | grep -q " build"; then
            echo "Running npm run build..."
            npm run build || echo "build failed but continuing"
          else
            echo "No build script found in package.json"
          fi

          echo "Build workspace:"
          ls -la
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('My-Sonar') {
          sh '''
            # Prefer sonar-scanner on agent; if not present, show message and continue
            if command -v sonar-scanner >/dev/null 2>&1; then
              sonar-scanner \
                -Dsonar.projectKey=nodeapp \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST} \
                -Dsonar.login=${SONAR_TOKEN}
            else
              echo "sonar-scanner not installed on agent. Configure sonar-scanner as a global tool or install on agent to run full scan."
            fi
          '''
        }
      }
    }

    stage('Create Artifact & Upload to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "Preparing artifact..."
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
          echo "Docker available: building image ${IMAGE_TAG}"
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
            echo "Logging into Docker registry..."
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            echo "Pushing ${IMAGE_TAG} ..."
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
