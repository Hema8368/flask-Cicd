pipeline {
  agent {
    docker {
      image 'node:20-bullseye'
      args  '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
      reuseNode true
    }
  }

  environment {
    COMPOSE_FILE = "ci/compose/docker-compose.ci.yml"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install Tools') {
      steps {
        sh '''
          set -eux
          apt-get update
          apt-get install -y docker.io curl

          curl -L "https://github.com/docker/compose/releases/download/v2.24.7/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
          chmod +x /usr/local/bin/docker-compose

          node -v
          npm -v
          docker --version
          docker-compose version
        '''
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          set -eux

          if [ -f apps/checkout/package.json ]; then
            cd apps/checkout
            npm ci
            npm test || true
            cd -
          fi

          if [ -f api/package.json ]; then
            cd api
            npm ci
            npm test || true
            cd -
          fi
        '''
      }
    }

    stage('Build & Deploy') {
      steps {
        sh '''
          set -eux
          docker-compose -f ${COMPOSE_FILE} down || true
          docker-compose -f ${COMPOSE_FILE} build --no-cache
          docker-compose -f ${COMPOSE_FILE} up -d

          chmod +x ci/scripts/wait-for-http.sh ci/scripts/smoke-test.sh
          ci/scripts/wait-for-http.sh http://localhost:8081/ 60
          ci/scripts/smoke-test.sh

          docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
          echo "OPEN UI: http://<EC2-PUBLIC-IP>:8081/"
        '''
      }
    }
  }

  post {
    always {
      sh 'echo "Pipeline finished"'
    }
  }
}
