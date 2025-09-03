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
          rem Se docker não estiver acessível, não falhe
          where docker >NUL 2>&1 || goto :done
          docker ps >NUL 2>&1 || goto :done

          docker rm -f %APP_CONT% %DB_CONT% 1>NUL 2>&1
          docker network rm %NET_NAME% 1>NUL 2>&1

          :done
          exit /b 0
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
          docker build -f db/Dockerfile.mysql -t %DB_IMAGE%  db
          if errorlevel 1 exit /b 1

          docker build -f app/Dockerfile.web-t %APP_IMAGE% app
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
          if (-not $ok) { Write-Host "DB não respondeu a tempo."; exit 1 }
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
              if ($r.StatusCode -in 200,301,302) { $ok = $true; break }
            } catch { }
            Start-Sleep -Seconds 2
          }
          if (-not $ok) { Write-Host "App não respondeu dentro do tempo."; exit 1 }
        '''
      }
    }
  }

  post {
    always {
      echo 'Coletando logs...'
      bat '''
        @echo off
        where docker >NUL 2>&1 || (echo (sem docker) > db-logs.txt & echo (sem docker) > app-logs.txt & exit /b 0)
        docker logs %DB_CONT%  > db-logs.txt  2>&1 || echo (sem logs DB)  > db-logs.txt
        docker logs %APP_CONT% > app-logs.txt 2>&1 || echo (sem logs APP) > app-logs.txt
        exit /b 0
      '''
      archiveArtifacts artifacts: '*.txt', onlyIfSuccessful: false
    }
    success { echo 'SUUUUUCESSO! App em: http://localhost:8200/' }
    failure { echo 'Eita. Falhou!' }
  }
}
