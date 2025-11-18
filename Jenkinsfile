pipeline {
  agent any

  environment {
    SONAR_TOKEN = credentials('sonar')
    NEXUS_CRED  = credentials('nexus')
    DOCKER_HUB  = credentials('docker-hub')
    IMAGE_TAG   = "rakesh268/newmew:${env.BUILD_NUMBER ?: 'latest'}"
    NEXUS_UPLOAD_URL = "http://51.20.37.198:8081/repository/nodejs/newmew.tar.gz"
    SONAR_HOST = "http://51.20.37.198:9000"
    NODE_IMAGE = "node:18"          // use non-alpine for better build tool compatibility
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
          echo "which node/npm/docker:"
          which node || true
          node -v || true
          npm -v || true
          which docker || echo "docker not found"
          echo "Jenkins UID/GID: $(id -u):$(id -g)"
        '''
      }
    }

    stage('Install & Build (auto-detect)') {
      steps {
        script {
          def nodeOnPath = sh(script: 'which node >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'
          def dockerOnPath = sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes'

          if (nodeOnPath) {
            echo "Using node from agent PATH"
            sh '''
              node -v
              npm -v
              npm ci --no-audit --no-fund
              if npm run | grep -q " build"; then npm run build || echo "build failed but continuing"; fi
              ls -la
            '''
          } else if (dockerOnPath) {
            // compute UID/GID for use inside docker so files are owned by jenkins user
            def uid = sh(script: 'id -u', returnStdout: true).trim()
            def gid = sh(script: 'id -g', returnStdout: true).trim()
            echo "No system node. Will run build inside Docker image ${env.NODE_IMAGE} as ${uid}:${gid}"

            // ensure a directory for npm cache exists on host to mount (reduce network)
            sh 'mkdir -p $HOME/.npm || true'
            // run docker with same uid:gid, mount workspace and npm cache, provide larger ulimit
            sh """
              docker run --rm \\
                -u ${uid}:${gid} \\
                -v "$WORKSPACE":"$WORKSPACE" \\
                -v "$HOME/.npm":"/home/node/.npm" \\
                -w "$WORKSPACE" \\
                --network host \\
                --ulimit nofile=8192:8192 \\
                ${NODE_IMAGE} /bin/bash -lc \\
                "set -e;
                 apt-get update >/dev/null 2>&1 || true;
                 # the node image already has node & npm; install build essentials if needed
                 if command -v apk >/dev/null 2>&1; then apk add --no-cache python3 make g++ || true; fi
                 if command -v apt-get >/dev/null 2>&1; then apt-get update >/dev/null 2>&1 || true; apt-get install -y --no-install-recommends python3 make g++ || true; fi
                 echo 'node:'; node -v;
                 echo 'npm:'; npm -v || true;
                 echo 'Running npm ci...';
                 npm ci --no-audit --no-fund;
                 if npm run | grep -q ' build'; then npm run build || echo 'build failed but continuing'; else echo 'No build script found'; fi;
                 echo 'Build complete. Listing workspace:'; ls -la"
            """
          } else {
            error """
No node/npm available and docker is not usable.
Fix options:
  - Install Node on this agent OR
  - Enable Docker for the Jenkins user OR
  - Configure the Jenkins NodeJS plugin and set a tool name and re-enable tools block.
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
              sonar-scanner -Dsonar.projectKey=nodeapp -Dsonar.sources=. -Dsonar.host.url=${SONAR_HOST} -Dsonar.login=${SONAR_TOKEN}
            """
          } else if (dockerOnPath) {
            echo "Running sonar-scanner in docker"
            sh """
              docker run --rm -v "$WORKSPACE":"$WORKSPACE" -w "$WORKSPACE" -e SONAR_LOGIN=${SONAR_TOKEN} sonarsource/sonar-scanner-cli:latest \\
                -Dsonar.projectKey=nodeapp -Dsonar.sources=. -Dsonar.host.url=${SONAR_HOST} -Dsonar.login=\$SONAR_LOGIN
            """
          } else {
            echo "Skipping SonarQube analysis: sonar-scanner not installed and docker not usable."
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
      when { expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' } }
      steps {
        sh '''
          echo "Building Docker image ${IMAGE_TAG}"
          docker build -t ${IMAGE_TAG} .
          docker images | head -n 20
        '''
      }
    }

    stage('Push Docker Image') {
      when { expression { return sh(script: 'which docker >/dev/null 2>&1 && echo yes || echo no', returnStdout: true).trim() == 'yes' } }
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
    always { echo "Pipeline finished!"; cleanWs() }
    success { echo "Pipeline succeeded." }
    failure { echo "Pipeline failed — check logs above." }
  }
}
