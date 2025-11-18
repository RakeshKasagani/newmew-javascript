pipeline {
  agent any

  tools {
    // Requires NodeJS plugin and a configured tool named 'node-18'
    nodejs 'node-18'
  }

  environment {
    // Credentials configured in Jenkins
    SONAR_TOKEN = credentials('sonar')          // secret text
    NEXUS_CRED  = credentials('nexus')          // username/password -> exposed as NEXUS_CRED_USR / NEXUS_CRED_PSW
    DOCKER_HUB  = credentials('docker-hub')     // username/password -> DOCKER_HUB_USR / DOCKER_HUB_PSW

    // URLs / image
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

    stage('Install Dependencies') {
      steps {
        sh '''
          echo "Node version: $(node -v)"
          echo "NPM version:  $(npm -v)"
          echo "Installing dependencies..."
          npm ci --no-audit --no-fund
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // Requires Sonar plugin config named 'My-Sonar' in Jenkins
        withSonarQubeEnv('My-Sonar') {
          sh '''
            # use sonar-scanner binary available on the agent or in image
            sonar-scanner \
              -Dsonar.projectKey=newmew \
              -Dsonar.sources=. \
              -Dsonar.host.url=${SONARQUBE_URL} \
              -Dsonar.login=${SONAR_TOKEN}
          '''
        }
      }
    }

    stage('Package Artifact') {
      steps {
        sh '''
          # create package (exclude node_modules)
          zip -r nodeapp.zip . -x "node_modules/*"
          ls -lh nodeapp.zip
        '''
      }
    }

    stage('Upload to Nexus') {
      steps {
        sh '''
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
          echo "Building Docker image ${DOCKER_IMAGE}..."
          docker build -t ${DOCKER_IMAGE} .
        '''
      }
    }

    stage('Push Docker Image') {
      steps {
        sh '''
          echo "Logging in to Docker Hub..."
          echo $DOCKER_HUB_PSW | docker login -u $DOCKER_HUB_USR --password-stdin
          echo "Pushing ${DOCKER_IMAGE}..."
          docker push ${DOCKER_IMAGE}
        '''
      }
    }
  }

  post {
    always {
      echo "Pipeline finished! Status: ${currentBuild.currentResult}"
    }
    success {
      echo "Success: ${DOCKER_IMAGE} build #${VERSION}"
    }
    failure {
      echo "Build failed."
    }
  }
}
