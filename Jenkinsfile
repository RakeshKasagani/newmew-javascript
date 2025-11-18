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

    stage('Install & Build') {
      steps {
        sh '''
          echo "Node version:"
          node -v

          echo "NPM version:"
          npm -v

          echo "Installing Node dependencies (npm ci)..."
          npm ci --no-audit --no-fund

          if npm run | grep -q " build"; then
            echo "Running npm run build..."
            npm run build || echo "build failed but continuing"
          else
            echo "No build script found"
          fi
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('My-Sonar') {
          sh '''
            # Using sonar-scanner installed on agent (adjust path if different)
            if [ -x /opt/sonar-scanner/bin/sonar-scanner ]; then
              /opt/sonar-scanner/bin/sonar-scanner \
                -Dsonar.projectKey=nodeapp \
                -Dsonar.sources=. \
                -Dsonar.host.url=http://51.20.37.198:9000 \
                -Dsonar.login=$SONAR_TOKEN
            else
              echo "sonar-scanner not found at /opt/sonar-scanner/bin/sonar-scanner"
            fi
          '''
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
            tar --ignore-failed-read --warning=no-file-changed -czf newmew.tar.gz *

            echo "Uploading TAR to Nexus..."
            curl -v -u $NEXUS_USER:$NEXUS_PASS --upload-file newmew.tar.gz http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz
          '''
        }
      }
    }

    stage('Build Docker Image (optional)') {
      when {
        expression {
          // only run if docker command exists on agent
          return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
        }
      }
      steps {
        sh '''
          echo "Building Docker image..."
          docker build -t rakesh268/newmew:latest .
        '''
      }
    }

    stage('Push Docker Image (optional)') {
      when {
        expression {
          return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push rakesh268/newmew:latest
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
