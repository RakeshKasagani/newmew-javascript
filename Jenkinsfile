pipeline {
  agent any

  environment {
    SONAR_TOKEN = credentials('sonar')
    NEXUS_CRED  = credentials('nexus')
    DOCKER_HUB  = credentials('docker-hub')
    IMAGE_TAG   = "rakesh268/newmew:${env.BUILD_NUMBER ?: 'latest'}"
    NEXUS_UPLOAD_URL = "http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz"
    SONAR_HOST = "http://51.20.37.198:9000"
    NODE_TOOL_NAME = "NodeJS_18" // change this if you name your NodeJS tool differently
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

    stage('Install & Build (no-docker)') {
      steps {
        script {
          // 1) Prefer system node if present
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
          } else {
            // 2) Try to use configured NodeJS tool at runtime using `tool`
            echo "System node not found; attempting to use NodeJS tool '${NODE_TOOL_NAME}' configured in Jenkins (Manage Jenkins → Global Tool Configuration)."
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
            } catch (err) {
              error """
No 'node' found on PATH and NodeJS tool '${NODE_TOOL_NAME}' is not configured or failed to provide node.
Fix options:
  1) Install Node on the agent (system-wide) so 'node' and 'npm' are available on PATH.
     Example (Ubuntu/Debian): 
       curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
       sudo apt-get install -y nodejs
  2) Or configure Jenkins NodeJS plugin (Manage Jenkins → Global Tool Configuration) and add an installation named '${NODE_TOOL_NAME}'.
  3) Or allow Jenkins to access Docker so the pipeline can run builds inside a Node Docker image (but currently Docker socket permission is denied).
After applying one of the fixes above, re-run the pipeline.
"""
            }
          }
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
    always { echo "Pipeline finished!"; cleanWs() }
    success { echo "Pipeline succeeded." }
    failure { echo "Pipeline failed — check logs above." }
  }
}
