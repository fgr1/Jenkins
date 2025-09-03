pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    DB_IMAGE   = 'devops/mysql-seeded:8.0'
    APP_IMAGE  = 'devops/flask-mysql:latest'

    NET_NAME   = 'devops-net'
    DB_CONT    = 'devops_db'
    APP_CONT   = 'devops_web'

    DB_PORT    = '3306:3306'
    APP_PORT   = '8200:8200'

    MYSQL_ROOT_PASSWORD = 'rootpass'
    MYSQL_USER          = 'appuser'
    MYSQL_PASSWORD      = 'apppass'
    MYSQL_DBNAME        = 'docker_e_kubernetes'
    MYSQL_ADDRESS       = 'devops_db:3306'
  }

  stages {
    stage('Cleanup') {
      steps {
        echo "Limpando workspace e recursos Docker..."
        deleteDir()
        sh '''
          set +e
          docker rm -f ${APP_CONT} ${DB_CONT} 2>/dev/null || true
          docker network rm ${NET_NAME} 2>/dev/null || true
          docker image prune -f || true
          set -e
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Construção') {
      steps {
        echo "Build das imagens (MySQL + App)..."
        sh '''
          docker build -t ${DB_IMAGE}  docker/mysql
          docker build -t ${APP_IMAGE} docker/app
          docker image ls | grep -E "demo/mysql-seeded|demo/flask-mysql" || true
        '''
      }
    }

    stage('Entrega') {
      steps {
        echo "Criando rede e subindo containers..."
        sh '''
          # Rede
          docker network create ${NET_NAME} || true

          # DB
          docker run -d --name ${DB_CONT} --network ${NET_NAME} -p ${DB_PORT} \
            -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
            -e MYSQL_DATABASE=${MYSQL_DBNAME} \
            -e MYSQL_USER=${MYSQL_USER} \
            -e MYSQL_PASSWORD=${MYSQL_PASSWORD} \
            ${DB_IMAGE}

          echo "Aguardando MySQL responder..."
          for i in $(seq 1 60); do
            if docker exec ${DB_CONT} mysqladmin ping -h 127.0.0.1 -p${MYSQL_ROOT_PASSWORD} --silent; then
              echo "MySQL OK"; break
            fi
            sleep 1
            if [ "$i" -eq 60 ]; then
              echo "Alguma coisa errada não está certa com o DB."
              docker logs ${DB_CONT} || true
              exit 1
            fi
          done

          # App
          docker run -d --name ${APP_CONT} --network ${NET_NAME} -p ${APP_PORT} \
            -e MYSQL_USERNAME=${MYSQL_USER} \
            -e MYSQL_PASSWORD=${MYSQL_PASSWORD} \
            -e MYSQL_ADDRESS=${MYSQL_ADDRESS} \
            -e MYSQL_DBNAME=${MYSQL_DBNAME} \
            ${APP_IMAGE}

          echo "Smoke test do app (HTTP 200 esperado)..."
          for i in $(seq 1 30); do
            if curl -sSf http://localhost:8200/ > /dev/null; then
              echo "Sucesso."; exit 0
            fi
            sleep 2
          done
          echo "Eita"
          docker logs ${APP_CONT} || true
          exit 1
        '''
      }
    }
  }

  post {
    always {
      echo "Coletando logs..."
      sh '''
        set +e
        docker logs ${DB_CONT}    > db-logs.txt 2>&1 || true
        docker logs ${APP_CONT}   > app-logs.txt 2>&1 || true
        set -e
      '''
      archiveArtifacts artifacts: '*.txt', onlyIfSuccessful: false
    }
    success {
      echo "Dale! App disponível em http://<host>:8200/"
    }
    failure {
      echo "Vixe! Algo deu errado."
    }
  }
}
