pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    APP_IMAGE = "demo/flask-mysql:latest"
    DB_IMAGE  = "demo/mysql-seeded:8.0"
    COMPOSE_FILE = "docker-compose.yml"
  }

  stages {
    stage('Cleanup') {
      steps {
        echo "Limpando workspace e imagens"
        deleteDir()
        sh '''
          docker system prune -af  || true
          docker volume prune -f   || true
          docker network prune -f  || true
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Construção') {
      steps {
        echo "Build das imagens"
        sh '''
          docker compose -f ${COMPOSE_FILE} build --no-cache
          docker image ls | grep -E "demo/flask-mysql|demo/mysql-seeded" || true
        '''
      }
    }

    stage('Entrega') {
      steps {
        echo "Subindo o ambiente com Docker Compose..."
        sh '''
          docker compose -f ${COMPOSE_FILE} up -d
          echo "Aguardando saúde do serviço web..."
          # Simples verificação de porta/HTTP
          for i in {1..30}; do
            curl -sSf http://localhost:8200/ > /dev/null && exit 0
            sleep 2
          done
          echo "Aplicação não respondeu a tempo." >&2
          docker compose logs --no-color > compose-logs.txt || true
          exit 1
        '''
      }
    }
  }

  post {
    always {
      echo "Coletando logs do Compose"
      sh 'docker compose logs --no-color > compose-logs.txt || true'
      archiveArtifacts artifacts: 'compose-logs.txt', onlyIfSuccessful: false
    }
    success {
      echo "Pipeline finalizada com sucesso!"
    }
    failure {
      echo "Falha no pipeline."
    }
  }
}
