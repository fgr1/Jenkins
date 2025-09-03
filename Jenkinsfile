pipeline {
  agent any

  options {
    timestamps()
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

    MYSQL_ROOT_PASSWORD = 'admin'
    MYSQL_USER          = 'user'
    MYSQL_PASSWORD      = 'password'
    MYSQL_DBNAME        = 'docker_e_kubernetes'
    MYSQL_ADDRESS       = 'devops_db:3306'
  }

  stages {
    stage('Cleanup') {
      steps {
        echo 'Limpando containers/rede...'
        bat '''
          @echo off
          docker rm -f %APP_CONT% %DB_CONT% 1>NUL 2>&1
          docker network rm %NET_NAME% 1>NUL 2>&1
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
        bat 'dir'
      }
    }

    stage('Construção') {
      steps {
        echo 'Build das imagens...'
        bat '''
          @echo off
          docker build -t %DB_IMAGE%  mysql
          if errorlevel 1 exit /b 1

          docker build -t %APP_IMAGE% app
          if errorlevel 1 exit /b 1

          docker images
        '''
      }
    }

    stage('Entrega') {
      steps {
        echo 'Criando rede, subindo DB e App...'

        bat '''
          @echo off
          docker network inspect %NET_NAME% 1>NUL 2>&1 || docker network create %NET_NAME%
          if errorlevel 1 exit /b 1
        '''

        bat '''
          @echo off
          docker run -d --name %DB_CONT% --network %NET_NAME% -p %DB_PORT% ^
            -e MYSQL_ROOT_PASSWORD=%MYSQL_ROOT_PASSWORD% ^
            -e MYSQL_DATABASE=%MYSQL_DBNAME% ^
            -e MYSQL_USER=%MYSQL_USER% ^
            -e MYSQL_PASSWORD=%MYSQL_PASSWORD% ^
            %DB_IMAGE%
          if errorlevel 1 exit /b 1
        '''

        powershell '''
          $ok = $false
          for ($i=0; $i -lt 60; $i++) {
            $p = Start-Process docker -ArgumentList "exec", "$env:DB_CONT", "mysqladmin","ping","-h","127.0.0.1","-p$env:MYSQL_ROOT_PASSWORD","--silent" -NoNewWindow -PassThru -Wait
            if ($p.ExitCode -eq 0) { $ok = $true; break }
            Start-Sleep -Seconds 1
          }
          if (-not $ok) {
            Write-Host "O bixo pegou, o DB não responde."; 
            exit 1
          }
        '''

        bat '''
          @echo off
          docker run -d --name %APP_CONT% --network %NET_NAME% -p %APP_PORT% ^
            -e MYSQL_USERNAME=%MYSQL_USER% ^
            -e MYSQL_PASSWORD=%MYSQL_PASSWORD% ^
            -e MYSQL_ADDRESS=%MYSQL_ADDRESS% ^
            -e MYSQL_DBNAME=%MYSQL_DBNAME% ^
            %APP_IMAGE%
          if errorlevel 1 exit /b 1
        '''

        powershell '''
          $ok = $false
          for ($i=0; $i -lt 30; $i++) {
            try {
              $r = Invoke-WebRequest -Uri "http://localhost:8200/" -UseBasicParsing -TimeoutSec 5
              if ($r.StatusCode -eq 200 -or $r.StatusCode -eq 302 -or $r.StatusCode -eq 301) { $ok = $true; break }
            } catch { }
            Start-Sleep -Seconds 2
          }
          if (-not $ok) {
            Write-Host "Alguma coisa errada não está certa"
            exit 1
          }
        '''
      }
    }
  }

  post {
    always {
      echo 'Coletando logs...'
      bat '''
        @echo off
        docker logs %DB_CONT%  > db-logs.txt  2>&1
        docker logs %APP_CONT% > app-logs.txt 2>&1
      '''
      archiveArtifacts artifacts: '*.txt', onlyIfSuccessful: false
    }
    success {
      echo 'Suuuucesso! App em: http://localhost:8200/'
    }
    failure {
      echo 'Eita, deu erro!'
    }
  }
}
