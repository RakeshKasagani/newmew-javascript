pipeline {
  agent any

  environment {
    // credentials configured in Jenkins credentials store
    SONAR_TOKEN = credentials('sonar')           // secret text
    NEXUS_CRED  = credentials('nexus')           // username/password -> NEXUS_CRED_USR / NEXUS_CRED_PSW
    DOCKER_HUB  = credentials('docker-hub')      // username/password -> DOCKER_HUB_USR / DOCKER_HUB_PSW

    SONARQUBE_URL = 'http://51.20.37.198:9000/'
    NEXUS_URL     = 'http://51.20.37.198:8081/'
    NEXUS_NPM_REPO = 'npm-hosted'                // <- change this if your Nexus npm repo has a different name
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
          # Write wrapper so subsequent steps can source nvm/node
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
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi
          echo "Node: $(node -v || true)"
          echo "NPM:  $(npm -v || true)"
          echo "Installing dependencies..."
          npm ci --no-audit --no-fund
        '''
      }
    }

    stage('Prepare package.json & npmignore') {
      steps {
        // Remove private field and bump version if it's 0.0.0; create .npmignore to exclude large assets
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi

          if [ -f package.json ]; then
            echo "Original package.json:"
            jq '.' package.json || cat package.json

            # Remove "private" if present
            node -e "let fs=require('fs'); let p=JSON.parse(fs.readFileSync('package.json')); if(p.private){ delete p.private; console.log('Removed private flag'); } if(p.version === '0.0.0' || !p.version){ p.version = '0.0.' + (process.env.BUILD_NUMBER || Date.now()); console.log('Bumped version to', p.version);} fs.writeFileSync('package.json', JSON.stringify(p,null,2));"

            echo "Updated package.json:"
            jq '.' package.json || cat package.json
          else
            echo "No package.json found!"
            exit 1
          fi

          # Create a conservative .npmignore to avoid publishing large assets
          cat > .npmignore <<'EOF'
# exclude large assets and CI files from published package
src/assets/*.png
src/assets/*.jpg
src/assets/*.jpeg
src/assets/*.otf
src/assets/*.ttf
public/
.scannerwork/
Dockerfile
Jenkinsfile
.git
node_modules/
EOF

          echo ".npmignore created (will reduce package size)."
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi

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
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi
          echo "Creating npm package tarball with npm pack..."
          npm pack
          echo "Pack created:"
          ls -lh *.tgz || true
        '''
      }
    }

    stage('Verify Nexus & Debug') {
      steps {
        // Lightweight check so we get clearer logs if the repo name is wrong or unreachable
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi

          NEXUS_REGISTRY="${NEXUS_URL%/}/repository/${NEXUS_NPM_REPO}/"
          echo "Checking Nexus registry URL: ${NEXUS_REGISTRY}"
          echo "curl -I output:"
          curl -I --max-time 10 "${NEXUS_REGISTRY}" || true

          echo "Showing the top of the generated .npmignore (for verification):"
          sed -n '1,120p' .npmignore || true

          echo "If the previous curl returned 404, confirm Nexus repo name and that it's a 'hosted' npm repo (not proxy/group)."
        '''
      }
    }

    stage('Publish to Nexus (npm)') {
      steps {
        // Publish the npm tarball to Nexus npm repository using npm publish (correct method for npm repos)
        sh '''
          set -e
          if [ -f .nvmrc_for_jenkins ]; then . ./.nvmrc_for_jenkins; fi

          TARBALL=$(ls *.tgz | head -n1)
          if [ -z "$TARBALL" ]; then
            echo "ERROR: no tarball found to publish"
            exit 1
          fi
          echo "Publishing ${TARBALL} to Nexus npm repository..."

          # Build registry URL and host-only (used to create .npmrc)
          NEXUS_REGISTRY="${NEXUS_URL%/}/repository/${NEXUS_NPM_REPO}/"
          HOST_ONLY=$(echo "${NEXUS_REGISTRY}" | sed -E 's#https?://##; s#/$##')

          # Quick reachability check (fail fast with readable message)
          STATUS=$(curl -o /dev/null -s -w "%{http_code}" --max-time 10 "${NEXUS_REGISTRY}" || echo "000")
          if [ "$STATUS" = "404" ]; then
            echo "ERROR: Nexus registry returned 404 - repository may not exist or name is wrong: ${NEXUS_REGISTRY}"
            echo "Please confirm repository '${NEXUS_NPM_REPO}' exists and is a hosted npm repo."
            exit 1
          fi

          # Use the Jenkins-provided credential variables NEXUS_CRED_USR / NEXUS_CRED_PSW
          # Create basic auth in base64 for .npmrc. We DO NOT print auth.
          AUTH_B64=$(printf '%s' "${NEXUS_CRED_USR}:${NEXUS_CRED_PSW}" | base64 --wrap=0 2>/dev/null || printf '%s' "${NEXUS_CRED_USR}:${NEXUS_CRED_PSW}" | openssl base64 -A)

          # Create a temporary .npmrc_for_publish (deleted after publish). Don't echo secret contents.
          cat > .npmrc_for_publish <<EOF
registry=${NEXUS_REGISTRY}
//${HOST_ONLY}/:_auth=${AUTH_B64}
//${HOST_ONLY}/:always-auth=true
EOF

          export npm_config_userconfig=$(pwd)/.npmrc_for_publish

          # Attempt publish. If registry rejects, npm will provide useful error in console.
          npm publish "$TARBALL" --registry "${NEXUS_REGISTRY}" --tag latest

          # cleanup
          rm -f .npmrc_for_publish
          echo "Publish completed."
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          if command -v docker >/dev/null 2>&1 && docker info >/dev/null 2>&1; then
            echo "Docker is available — building image ${DOCKER_IMAGE}..."
            docker build -t ${DOCKER_IMAGE} .
          else
            echo "Docker not available on this agent or cannot access daemon — skipping Docker build."
            exit 0
          fi
        '''
      }
    }

    stage('Push Docker Image') {
      steps {
        sh '''
          set -e
          if command -v docker >/dev/null 2>&1 && docker info >/dev/null 2>&1; then
            echo "Logging into Docker Hub..."
            echo $DOCKER_HUB_PSW | docker login -u $DOCKER_HUB_USR --password-stdin
            echo "Pushing ${DOCKER_IMAGE}..."
            docker push ${DOCKER_IMAGE}
          else
            echo "Docker not available or daemon not accessible — skipping Docker push."
            exit 0
          fi
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
