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
                    rem Ignora erros se o docker não estiver rodando
                    docker ps >NUL 2>&1 || exit /b 0
                    docker rm -fv %APP_CONT% %DB_CONT% >NUL 2>&1
                    docker network rm %NET_NAME% >NUL 2>&1
                    exit /b 0
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
                echo 'Corrigindo arquivos no workspace e iniciando o build...'

                powershell "(Get-Content db/codigo.sql) -replace 'CREATE DATABASE docker_e_kubernetes;', '' | Set-Content db/codigo.sql"

                powershell "(Get-Content app/main.py) -replace 'class Usuario\\(Base\\):', 'class Usuario(db.Model):' | Set-Content app/main.py"
                
                bat '''
                    @echo on
                    docker build -f db/Dockerfile.mysql -t %DB_IMAGE% db
                    if errorlevel 1 exit /b 1

                    docker build -f app/Dockerfile.web -t %APP_IMAGE% app
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Entrega') {
            steps {
                echo 'Criando rede e subindo os containers...'

                bat '''
                    @echo off
                    docker network create %NET_NAME% >NUL 2>&1

                    docker run -d --name %DB_CONT% --network %NET_NAME% -p %DB_PORT% ^
                        -e MYSQL_ROOT_PASSWORD=%MYSQL_ROOT_PASSWORD% ^
                        -e MYSQL_DATABASE=%MYSQL_DBNAME% ^
                        -e MYSQL_USER=%MYSQL_USER% ^
                        -e MYSQL_PASSWORD=%MYSQL_PASSWORD% ^
                        %DB_IMAGE%
                    if errorlevel 1 exit /b 1
                '''

                echo 'Aguardando o banco de dados...'
                powershell '''
                    for ($i=0; $i -lt 60; $i++) {
                        $p = docker exec $env:DB_CONT mysqladmin ping -h 127.0.0.1 -p$env:MYSQL_ROOT_PASSWORD --silent
                        if ($LASTEXITCODE -eq 0) {
                            Write-Host "Banco de dados está pronto."
                            exit 0
                        }
                        Start-Sleep -Seconds 1
                    }
                    Write-Host "Erro: Banco de dados não respondeu a tempo."
                    exit 1
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

                echo 'Aguardando a aplicação responder...'
                powershell '''
                    for ($i=0; $i -lt 30; $i++) {
                        try {
                            $r = Invoke-WebRequest -Uri "http://localhost:8200/" -UseBasicParsing
                            if ($r.StatusCode -eq 200) {
                                Write-Host "Aplicação está pronta."
                                exit 0
                            }
                        } catch {}
                        Start-Sleep -Seconds 2
                    }
                    Write-Host "Erro: Aplicação não respondeu a tempo."
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            echo 'Coletando logs dos containers...'
            bat '''
                @echo off
                docker logs %DB_CONT% > db-logs.txt 2>&1
                docker logs %APP_CONT% > app-logs.txt 2>&1
                exit /b 0
            '''
            archiveArtifacts artifacts: '*.txt', onlyIfSuccessful: false
        }
        success {
            echo 'SUUUUUCESSO! Aplicação em: http://localhost:8200/'
        }
        failure {
            echo 'EITA! Alguma coisa errada não está certa!'
        }
    }
}