pipeline {
  agent any

  environment {
    // credentials configured in Jenkins credentials store
    SONAR_TOKEN = credentials('sonar')           // secret text
    NEXUS_CRED  = credentials('nexus')           // username/password -> NEXUS_CRED_USR / NEXUS_CRED_PSW
    DOCKER_HUB  = credentials('docker-hub')      // username/password -> DOCKER_HUB_USR / DOCKER_HUB_PSW

    SONARQUBE_URL = 'http://13.51.204.28:9000'
    NEXUS_URL     = 'http://13.51.204.28:8081'
    DOCKER_IMAGE  = "rakesh268/newmew:latest"
    VERSION       = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'master', url: 'https://github.com/RakeshKasagani/newmew-javascript.git'
      }
    }

    stage('Ensure Node') {
      steps {
        // Install nvm + node 18 if node not available. This is idempotent.
        sh '''
          set -e
          if command -v node >/dev/null 2>&1; then
            echo "Node already installed: $(node -v)"
          else
            echo "Node not found — installing nvm + Node 18..."
            export NVM_DIR="$HOME/.nvm"
            if [ ! -s "$NVM_DIR/nvm.sh" ]; then
              curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
            fi
            . "$NVM_DIR/nvm.sh"
            nvm install 18
            nvm use 18
            echo "Installed Node: $(node -v)"
          fi
          # Ensure nvm/node are available to subsequent steps in this shell; for declarative pipeline each sh runs in a fresh shell,
          # so we write a wrapper file that other steps can source.
          echo 'export NVM_DIR="$HOME/.nvm"' > .nvmrc_for_jenkins
          echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"' >> .nvmrc_for_jenkins
          echo 'nvm use 18 >/dev/null 2>&1 || true' >> .nvmrc_for_jenkins
          chmod +x .nvmrc_for_jenkins
          echo "Wrote .nvmrc_for_jenkins - source it in later steps."
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        // Source the nvm wrapper so node/npm are available in this shell
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then
            . ./.nvmrc_for_jenkins
          fi
          echo "Node: $(node -v || true)"
          echo "NPM:  $(npm -v || true)"
          echo "Installing dependencies..."
          npm ci --no-audit --no-fund
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // Use the sonar-scanner npm package via npx so you don't need system sonar-scanner.
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi

          # install sonar-scanner locally if not present (will be cached in workspace)
          if ! npx --no-install sonar-scanner --version >/dev/null 2>&1; then
            echo "Installing sonar-scanner npm package..."
            npm install --no-audit --no-fund sonar-scanner
          fi

          echo "Running sonar-scanner via npx..."
          npx sonar-scanner \
            -Dsonar.projectKey=newmew \
            -Dsonar.sources=. \
            -Dsonar.host.url=${SONARQUBE_URL} \
            -Dsonar.login=${SONAR_TOKEN}
        '''
      }
    }

    stage('Package Artifact') {
      steps {
        sh '''
          set -e
          zip -r nodeapp.zip . -x "node_modules/*" ".git/*"
          ls -lh nodeapp.zip
        '''
      }
    }

    stage('Upload to Nexus') {
      steps {
        sh '''
          set -e
          echo "Uploading nodeapp.zip to Nexus..."
          curl -v -u $NEXUS_CRED_USR:$NEXUS_CRED_PSW \
            --upload-file nodeapp.zip \
            ${NEXUS_URL}/repository/nodejs/newmew.zip
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          echo "Building Docker image ${DOCKER_IMAGE}..."
          docker build -t ${DOCKER_IMAGE} .
        '''
      }
    }

    stage('Push Docker Image') {
      steps {
        sh '''
          set -e
          echo "Logging into Docker Hub..."
          echo $DOCKER_HUB_PSW | docker login -u $DOCKER_HUB_USR --password-stdin
          echo "Pushing ${DOCKER_IMAGE}..."
          docker push ${DOCKER_IMAGE}
        '''
      }
    }
  }

  post {
    always {
      echo "Pipeline finished! status=${currentBuild.currentResult}"
    }
    success {
      echo "Success: ${DOCKER_IMAGE} build #${VERSION}"
    }
    failure {
      echo "Build FAILED."
    }
  }
}
