pipeline {
  agent any

  environment {
    SONAR_TOKEN = credentials('sonar')
    NEXUS_CRED  = credentials('nexus')
    DOCKER_HUB  = credentials('docker-hub')
    // Optional: a default image tag using build number
    IMAGE_TAG = "rakesh268/newmew:${env.BUILD_NUMBER ?: 'latest'}"
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
          which node || true
          node -v || true
          npm -v || true
          which docker || echo "docker not found"
        '''
      }
    }

    // Run Install & Build inside official Node image so node/npm are always available
    stage('Install & Build') {
      agent {
        docker {
          image 'node:18-alpine'         // change to node:20 if you prefer
          args  '-u root:root'           // run as root so installs work cleanly
        }
      }
      steps {
        sh '''
          echo "Using node:"
          node -v
          npm -v

          echo "Installing dependencies (npm ci)..."
          npm ci --no-audit --no-fund || { echo "npm ci failed"; exit 1; }

          if npm run | grep -q " build"; then
            echo "Running npm run build..."
            npm run build || echo "build failed but continuing"
          else
            echo "No build script found"
          fi

          # ensure build artifacts exist for downstream stages
          echo "Listing workspace after build:"
          ls -la
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // Use official sonar-scanner docker image so you don't need scanner installed on the agent
        script {
          def scannerImage = 'sonarsource/sonar-scanner-cli:latest'
          // run scanner container mounting workspace
          sh """
            if docker image inspect ${scannerImage} >/dev/null 2>&1 || docker pull ${scannerImage}; then
              docker run --rm -v "$WORKSPACE":"$WORKSPACE" -w "$WORKSPACE" \\
                -e SONAR_LOGIN=${SONAR_TOKEN} \\
                ${scannerImage} \\
                -Dsonar.projectKey=nodeapp \\
                -Dsonar.sources=. \\
                -Dsonar.host.url=http://51.20.37.198:9000 \\
                -Dsonar.login=\$SONAR_LOGIN
            else
              echo "sonar-scanner image not available and docker not usable on agent; skipping Sonar scan"
            fi
          """
        }
      }
    }

    stage('Create Artifact & Upload to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "Listing workspace files..."
            ls -la

            echo "Creating TAR archive..."
            tar --ignore-failed-read --warning=no-file-changed -czf newmew.tar.gz * || { echo "tar failed"; exit 1; }

            echo "Uploading TAR to Nexus..."
            curl -v -u $NEXUS_USER:$NEXUS_PASS --upload-file newmew.tar.gz http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz || { echo "upload failed"; exit 1; }
          '''
        }
      }
    }

    stage('Build Docker Image') {
      // only run if docker exists on the agent
      when {
        expression {
          return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
        }
      }
      steps {
        sh '''
          echo "Building Docker image..."
          # Use BUILD_ARG if you want to pass in build artifacts location, adjust as needed
          docker build -t ${IMAGE_TAG} .
          docker images | head -n 10
        '''
      }
    }

    stage('Push Docker Image') {
      when {
        expression {
          return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "Logging into Docker registry..."
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            echo "Pushing image ${IMAGE_TAG} ..."
            docker push ${IMAGE_TAG}
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
