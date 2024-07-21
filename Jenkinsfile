pipeline {
  agent any
  tools {
    nodejs 'node'
  }
  environment {
    GITHUB_CREDENTIALS = credentials('GITHUB_CREDENTIALS') 
    DOCKER_CREDENTIALS = credentials('DOCKER_CREDENTIALS')
    autoscaler_registry = 'dongrep/eks-autoscaler'
  }
  stages {
    stage('Clone repository') {
      steps {
        cleanWs()
        checkout scm
      }
    }
    stage('Validate Conventional Commits') {
      steps {
        withEnv(["GITHUB_TOKEN=${GITHUB_CREDENTIALS_PSW}"]) {
          sh '''
          echo "Validate commit messages"

          npm i -D @semantic-release/commit-analyzer @semantic-release/exec @semantic-release/git @semantic-release/github
          npx semantic-release --dry-run
          '''
        }
      }
    }
    stage('Checks for helm') {
      steps {
        withEnv(["GITHUB_TOKEN=${GITHUB_CREDENTIALS_PSW}"]) {
          sh '''
          echo "Checks for helm"
          helm lint .
          '''
        }
      }
    }
    stage('Change chart version and release') {
      when {
        branch 'main'
      }
      steps {
        withEnv(["GITHUB_TOKEN=${GITHUB_CREDENTIALS_PSW}"]) {
          sh '''
          npx semantic-release --dry-run > semantic-release-output.txt
          newVersion=$(grep 'next release version' semantic-release-output.txt | awk '{print $NF}')
          echo "Change chart version"
          echo "New version: ${newVersion}"
          sed -i "s/^version: .*/version: ${newVersion}/" Chart.yaml
          git add Chart.yaml
          npx semantic-release
          '''
        }
      }
    }
    stage('Build and Push Autoscaler Image') {
      when {
        branch 'main'
      }
      steps {
        script {
          // Login to Docker Hub
          echo 'Login to Docker Hub'
          sh 'echo $DOCKER_CREDENTIALS_PSW | docker login -u $DOCKER_CREDENTIALS_USR --password-stdin'

          // Create a builder instance if not exists
          echo 'Ensure buildx builder instance'
          sh '''
              docker buildx create --use --name mybuilder || true
              docker buildx inspect mybuilder --bootstrap || true
          '''

          // Build and push multi-architecture image
          echo 'Build and push multi-architecture image'
          sh """
              docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
                  --progress=plain \
                  --cache-from=type=registry,ref=${autoscaler_registry}:cache \
                  --cache-to=type=inline \
                  -t ${autoscaler_registry}:`${newVersion}` \
                  -t ${autoscaler_registry}:latest \
                  -f ./Dockerfile.webapp \
                  --push .
          """
        }
      }
    }
  }
  post {
    always {
      cleanWs()
      sh 'docker system prune -a -f'
      sh 'docker logout'
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed!'
    }
  }
}
