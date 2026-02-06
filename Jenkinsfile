pipeline {
    agent any

    tools {
        nodejs 'Node_js'
        jdk 'JDK_HOME'
    }

    environment {
        ANGULAR_DIR = 'Angular/SimpleClient'
        DOTNET_DIR  = 'DotNet/SimpleAPI'
        OUT_DIR     = 'out\\SimpleAPI'
        TIMESTAMP  = new Date().format("yyyyMMdd_HHmmss", TimeZone.getTimeZone('UTC'))
        BACKUP_DIR = "C:\\inetpub\\backups\\${TIMESTAMP}_build_${BUILD_NUMBER}"
        BACKUP_ZIP = "C:\\inetpub\\backups\\SimpleApp_backup_${TIMESTAMP}_build_${BUILD_NUMBER}.zip"
    }
    

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/shohail-DeV/Angular-Dotnet.git'
            }
        }

        stage('Build Angular') {
            steps {
                dir(env.ANGULAR_DIR) {
                    bat 'npm install --legacy-peer-deps'
                    bat 'npm run build -- --configuration production'
                }
            }
        }

        stage('Build .NET API') {
            steps {
                dir(env.DOTNET_DIR) {
                    bat 'dotnet restore'
                    bat 'dotnet build -c Release'
                    bat "dotnet publish SimpleAPI.csproj -c Release -o ..\\..\\${OUT_DIR}"
                }
            }
        }

        stage('Security Scan - Semgrep (SAST)') {
    steps {
        bat '''
        echo === SEMGREP SAST SCAN START ===

        REM --- Windows + Python stability ---
        set PYTHONUTF8=1
        set PYTHONIOENCODING=utf-8
        set SEMGREP_DISABLE_GIT=1
        set SEMGREP_USE_GIT=0

        REM --- Install Semgrep (pinned for stability) ---
        pip install semgrep==1.151.0 || exit /b 1

        REM --- Run Semgrep with explicit, low-noise rules ---
        semgrep ^
  --config p/security-audit ^
  --config p/owasp-top-ten ^
  --config p/csharp ^
  --config p/javascript ^
  --config p/secrets ^
  --severity ERROR ^
  --no-git-ignore ^
  --json ^
  --output semgrep.json ^
  Angular DotNet


        IF %ERRORLEVEL% NEQ 0 (
            echo ❌ High-severity security issues detected
            exit /b 1
        )

        echo === SEMGREP SAST SCAN PASSED ===
        '''
    }

    post {
        always {
            archiveArtifacts artifacts: 'semgrep.json', fingerprint: true
        }
        failure {
            echo '❌ Build blocked due to high-severity security findings'
        }
    }
}






        
        /* ================= BACKUP ================= */

        stage('Backup Current Production') {
            steps{
                bat """
                setlocal EnableDelayedExpansion
                echo === BACKUP START ===

                mkdir "${BACKUP_DIR}\\client" 2>nul
                mkdir "${BACKUP_DIR}\\api" 2>nul

                if exist "C:\\inetpub\\wwwroot\\SimpleClient" (
                    robocopy C:\\inetpub\\wwwroot\\SimpleClient "${BACKUP_DIR}\\client" /MIR
                    if !ERRORLEVEL! GTR 3 exit /b !ERRORLEVEL!
                )

                if exist "C:\\inetpub\\api\\SimpleAPI" (
                    robocopy C:\\inetpub\\api\\SimpleAPI "${BACKUP_DIR}\\api" /MIR
                    if !ERRORLEVEL! GTR 3 exit /b !ERRORLEVEL!
                )

                echo === ZIPPING BACKUP ===
                powershell -Command ^
                "Compress-Archive -Path '${BACKUP_DIR}\\*' -DestinationPath '${BACKUP_ZIP}' -Force"

                echo === CLEANUP RAW BACKUP FOLDER ===
                rmdir /s /q "${BACKUP_DIR}"

                echo Backup zip created at ${BACKUP_ZIP}
                exit /b 0
        """
                
            }               
}



        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts artifacts: '''
                    Angular/SimpleClient/dist/**,
                    out/SimpleAPI/**
                ''', fingerprint: true
            }
        }

        /* ================= DEPLOY + HEALTH + ROLLBACK ================= */

        stage('Deploy with Auto-Rollback') {
            steps {
                bat """
                setlocal EnableDelayedExpansion

                echo === STOP IIS ===
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

                echo === DEPLOY FILES ===
                robocopy Angular\\SimpleClient\\dist\\SimpleClient\\browser C:\\inetpub\\wwwroot\\SimpleClient /MIR
                if !ERRORLEVEL! GTR 3 goto ROLLBACK

                robocopy ${OUT_DIR} C:\\inetpub\\api\\SimpleAPI /MIR
                if !ERRORLEVEL! GTR 3 goto ROLLBACK

                echo === START IIS ===
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

                echo === HEALTH CHECK ===
                set RETRIES=10

                :CHECK
                ping 127.0.0.1 -n 5 > nul

                curl -s -o nul -w "%%{http_code}" http://localhost/health > status.txt
                set /p STATUS=<status.txt

                echo Health status: !STATUS!

                if "!STATUS!"=="200" goto SUCCESS

                set /a RETRIES-=1
                if !RETRIES! LEQ 0 goto ROLLBACK

                goto CHECK

                :SUCCESS
                echo API is healthy
                exit /b 0

                :ROLLBACK
                echo === ROLLBACK FROM ZIPPED BACKUP ===

                powershell -Command ^
                "Expand-Archive -Path '${BACKUP_ZIP}' -DestinationPath '${BACKUP_DIR}' -Force"

                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

                robocopy "${BACKUP_DIR}\\client" C:\\inetpub\\wwwroot\\SimpleClient /MIR
                robocopy "${BACKUP_DIR}\\api" C:\\inetpub\\api\\SimpleAPI /MIR

                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

                rmdir /s /q "${BACKUP_DIR}"
                exit /b 1

                :SUCCESS
                echo Deployment successful
                exit /b 0
                """
            }
        }
        
    }

    post {
        success {
            echo '✅ Deployment successful and backup preserved'
        }
        failure {
            echo '❌ Deployment failed — rollback executed from backup'
        }
    }
}
