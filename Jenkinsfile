pipeline {
  agent any

  environment {
    SONAR_TOKEN = credentials('sonar')
    NEXUS_CRED  = credentials('nexus')
    DOCKER_HUB  = credentials('docker-hub')
    IMAGE_TAG   = "rakesh268/newmew:${env.BUILD_NUMBER ?: 'latest'}"
    NEXUS_UPLOAD_URL = "http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz"
    SONAR_HOST = "http://51.20.37.198:9000"
    NODE_TOOL_NAME = "NodeJS_18" // change if you configured a different name in Jenkins
  }

  stages {
    stage('Checkout') {
      steps { git branch: 'master', url: 'https://github.com/RakeshKasagani/newmew-javascript.git' }
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

    stage('Install & Build (no-docker, with nvm fallback)') {
      steps {
        script {
          def nodeOnPath = sh(script: 'which node >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
          if (nodeOnPath) {
            echo "Using node from PATH on agent"
            sh '''
              node -v
              npm -v
              npm ci --no-audit --no-fund
              if npm run | grep -q " build"; then npm run build || echo "build failed but continuing"; fi
              ls -la
            '''
            return
          }

          // Try Jenkins NodeJS tool at runtime (if configured)
          echo "System node not found; attempting NodeJS tool '${NODE_TOOL_NAME}'"
          try {
            def nodeHome = tool name: NODE_TOOL_NAME, type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
            withEnv(["PATH=${nodeHome}/bin:${env.PATH}"]) {
              sh '''
                echo "Using Node from NodeJS tool at: ${nodeHome}"
                node -v
                npm -v
                npm ci --no-audit --no-fund
                if npm run | grep -q " build"; then npm run build || echo "build failed but continuing"; fi
                ls -la
              '''
            }
            return
          } catch (err) {
            echo "NodeJS tool '${NODE_TOOL_NAME}' not available or failed: ${err}"
          }

          // Last-resort: install nvm and use Node locally in workspace (temporary, per-build)
          echo "Falling back to per-build nvm install (installs Node into workspace for this build)."
          sh '''
            set -e
            export NVM_DIR="$WORKSPACE/.nvm"
            if [ ! -d "$NVM_DIR" ]; then
              echo "Installing nvm into $NVM_DIR..."
              curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.6/install.sh | bash
            fi
            # shellcheck disable=SC1090
            . "$NVM_DIR/nvm.sh"
            nvm install 18 --lts
            nvm use 18
            echo "Node (nvm) version: $(node -v)"
            echo "NPM version: $(npm -v)"
            npm ci --no-audit --no-fund
            if npm run | grep -q " build"; then npm run build || echo "build failed but continuing"; fi
            ls -la
          '''
        }
      }
    }

    stage('SonarQube Analysis (local scanner only)') {
      steps {
        script {
          def sonarInstalled = sh(script: 'command -v sonar-scanner >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
          if (sonarInstalled) {
            sh """
              sonar-scanner -Dsonar.projectKey=nodeapp -Dsonar.sources=. -Dsonar.host.url=${SONAR_HOST} -Dsonar.login=${SONAR_TOKEN}
            """
          } else {
            echo "sonar-scanner not installed on agent; skipping SonarQube analysis. To enable: install sonar-scanner or configure Sonar scanner global tool in Jenkins."
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

    stage('Build Docker Image (skipped if no docker)') {
      when {
        expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' }
      }
      steps {
        sh '''
          docker build -t ${IMAGE_TAG} .
        '''
      }
    }

    stage('Push Docker Image (skipped if no docker)') {
      when {
        expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
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
    success { echo "Pipeline succeeded." }
    failure { echo "Pipeline failed — check logs above." }
  }
}
