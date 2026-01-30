pipeline {
    agent any

    tools {
        nodejs 'Node_js'
        jdk 'JDK_HOME'
    }

    environment {
        ANGULAR_DIR = 'Angular/SimpleClient'
        DOTNET_DIR  = 'DotNet/SimpleAPI'
        BACKUP_DIR  = "C:\\inetpub\\backups\\${BUILD_NUMBER}"
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
                    bat 'dotnet publish SimpleAPI.csproj -c Release -o ..\\..\\out\\SimpleAPI'
                }
            }
        }

        // stage('SonarQube Analysis') {
        //     steps {
        //         withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
        //             withSonarQubeEnv('SonarQube-Server') {
        //                 script {
        //                     def scannerHome = tool 'SonarScanner'
        //                     bat """
        //                     "${scannerHome}\\bin\\sonar-scanner.bat" ^
        //                       -Dsonar.projectKey=Angular-DotNetCICD ^
        //                       -Dsonar.projectName=Angular-DotNetCICD ^
        //                       -Dsonar.sources=Angular/SimpleClient/src,DotNet/SimpleAPI ^
        //                       -Dsonar.exclusions=**/node_modules/**,**/bin/**,**/obj/** ^
        //                       -Dsonar.token=%SONAR_TOKEN%
        //                     """
        //                 }
        //             }
        //         }
        //     }
        // }

        // stage('Quality Gate (CE Workaround)') {
        //     steps {
        //         withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
        //             bat '''
        //             curl -u %SONAR_TOKEN%: ^
        //             "http://172.27.31.63:9000/api/qualitygates/project_status?projectKey=Angular-DotNetCICD" ^
        //             -o quality.json

        //             findstr /C:"ERROR" quality.json && echo Quality Gate Failed || echo Quality Gate Passed
        //             '''
        //         }
        //     }
        // }

                /* ================= BACKUP GATE ================= */

        stage('Backup Current Production') {
    steps {
        bat '''
        echo === BACKUP START ===

        mkdir "%BACKUP_DIR%\\client" 2>nul
        mkdir "%BACKUP_DIR%\\api" 2>nul

        if exist "C:\\inetpub\\wwwroot\\SimpleClient" (
            robocopy C:\\inetpub\\wwwroot\\SimpleClient "%BACKUP_DIR%\\client" /MIR
            set RC=%ERRORLEVEL%
            if %RC% LEQ 3 (exit /b 0) else (exit /b %RC%)
        )

        if exist "C:\\inetpub\\api\\SimpleAPI" (
            robocopy C:\\inetpub\\api\\SimpleAPI "%BACKUP_DIR%\\api" /MIR
            set RC=%ERRORLEVEL%
            if %RC% LEQ 3 (exit /b 0) else (exit /b %RC%)
        )

        echo Backup stored at %BACKUP_DIR%
        '''
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



        /* ================= DEPLOY + ROLLBACK ================= */

        stage('Deploy with Backup-Driven Rollback') {
            steps {
                bat '''
                setlocal ENABLEDELAYEDEXPANSION

                echo === STOP IIS ===
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

                echo === DEPLOY NEW VERSION ===
                robocopy Angular\\SimpleClient\\dist\\SimpleClient\\browser C:\\inetpub\\wwwroot\\SimpleClient /MIR
set RC=%ERRORLEVEL%
if %RC% GTR 3 goto ROLLBACK

robocopy out\\SimpleAPI C:\\inetpub\\api\\SimpleAPI /MIR
set RC=%ERRORLEVEL%
if %RC% GTR 3 goto ROLLBACK

                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

                echo === HEALTH CHECK ===
                set RETRIES=10

                :CHECK
                ping 127.0.0.1 -n 5 >nul
                for /f %%s in ('curl -o nul -s -w "%%{http_code}" http://localhost/api/health') do set STATUS=%%s

                if "%STATUS%"=="200" goto SUCCESS

                set /a RETRIES-=1
                if %RETRIES% LEQ 0 goto ROLLBACK
                goto CHECK

                :ROLLBACK
                echo === ROLLBACK FROM BACKUP ===

                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

                robocopy "%BACKUP_DIR%\\client" C:\\inetpub\\wwwroot\\SimpleClient /MIR
                robocopy "%BACKUP_DIR%\\api" C:\\inetpub\\api\\SimpleAPI /MIR

                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

                exit /b 1

                :SUCCESS
                echo Deployment successful
                exit /b 0
                '''
            }
        }

        stage('Archive Backup') {
            steps {
                archiveArtifacts artifacts: "C:/inetpub/backups/${BUILD_NUMBER}/**"
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


// pipeline {
//     agent any

//     tools {
//         nodejs 'Node_js'
//         jdk 'JDK_HOME'
//     }

//     environment {
//         ANGULAR_DIR = 'Angular/SimpleClient'
//         DOTNET_DIR  = 'DotNet/SimpleAPI'
//         BACKUP_DIR  = "C:\\inetpub\\backups\\${BUILD_NUMBER}"
//     }

//     stages {

//         stage('Checkout') {
//             steps {
//                 git branch: 'main',
//                     url: 'https://github.com/shohail-DeV/Angular-Dotnet.git'
//             }
//         }

//         stage('Build Angular') {
//             steps {
//                 dir(env.ANGULAR_DIR) {
//                     bat 'npm install --legacy-peer-deps'
//                     bat 'npm run build -- --configuration production'
//                 }
//             }
//         }

//         stage('Build .NET API') {
//             steps {
//                 dir(env.DOTNET_DIR) {
//                     bat 'dotnet restore'
//                     bat 'dotnet build -c Release'
//                     bat 'dotnet publish SimpleAPI.csproj -c Release -o ..\\..\\out\\SimpleAPI'
//                 }
//             }
//         }

//         stage('SonarQube Analysis') {
//             steps {
//                 withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
//                     withSonarQubeEnv('SonarQube-Server') {
//                         script {
//                             def scannerHome = tool 'SonarScanner'
//                             bat """
//                             "${scannerHome}\\bin\\sonar-scanner.bat" ^
//                               -Dsonar.projectKey=Angular-DotNetCICD ^
//                               -Dsonar.projectName=Angular-DotNetCICD ^
//                               -Dsonar.sources=Angular/SimpleClient/src,DotNet/SimpleAPI ^
//                               -Dsonar.exclusions=**/node_modules/**,**/bin/**,**/obj/**,**/publish/** ^
//                               -Dsonar.sourceEncoding=UTF-8 ^
//                               -Dsonar.token=%SONAR_TOKEN%
//                             """
//                         }
//                     }
//                 }
//             }
//         }

//         stage('Quality Gate (CE Workaround)') {
//             steps {
//                 withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
//                     bat '''
//                     curl -u %SONAR_TOKEN%: ^
//                     "http://172.27.31.63:9000/api/qualitygates/project_status?projectKey=Angular-DotNetCICD" ^
//                     -o quality.json

//                     findstr /C:"ERROR" quality.json && echo Quality Gate Failed || echo Quality Gate Passed
//                     '''
//                 }
//             }
//         }

//         stage('Archive Artifacts') {
//             steps {
//                 archiveArtifacts artifacts: '''
//                     Angular/SimpleClient/dist/**,
//                     out/SimpleAPI/**
//                 ''', fingerprint: true
//             }
//         }

//         stage('Zero-Downtime Deploy with Auto-Rollback') {
//     steps {
//         bat '''
//         setlocal ENABLEDELAYEDEXPANSION

//         REM ===== STOP IIS =====
//         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
//         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

//         REM ===== PREP =====
//         rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_new 2>nul
//         rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_new 2>nul

//         mkdir C:\\inetpub\\wwwroot\\SimpleClient_new
//         mkdir C:\\inetpub\\api\\SimpleAPI_new

//         REM ===== COPY =====
//         xcopy Angular\\SimpleClient\\dist\\SimpleClient\\browser ^
//               C:\\inetpub\\wwwroot\\SimpleClient_new /E /I /Y

//         robocopy out\\SimpleAPI C:\\inetpub\\api\\SimpleAPI_new /MIR /NFL /NDL /NP
//         IF %ERRORLEVEL% GTR 3 exit /b 1

//         REM ===== ATOMIC SWAP =====
//         rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_prev 2>nul
//         rename C:\\inetpub\\wwwroot\\SimpleClient SimpleClient_prev
//         rename C:\\inetpub\\wwwroot\\SimpleClient_new SimpleClient

//         rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_prev 2>nul
//         rename C:\\inetpub\\api\\SimpleAPI SimpleAPI_prev
//         rename C:\\inetpub\\api\\SimpleAPI_new SimpleAPI

//         REM ===== START IIS =====
//         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
//         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

//         REM ===== HEALTH CHECK WITH RETRIES =====
//         set RETRIES=10

//         :CHECK_LOOP
//         ping 127.0.0.1 -n 6 >nul

//         for /f %%s in ('curl -o nul -s -w "%%{http_code}" http://localhost/api/health') do set API_STATUS=%%s
//         echo Health check attempt %RETRIES% - Status %API_STATUS%

//         if "%API_STATUS%"=="200" goto SUCCESS

//         set /a RETRIES-=1
//         if %RETRIES% LEQ 0 goto ROLLBACK

//         goto CHECK_LOOP

//         :ROLLBACK
//         echo Health check FAILED. Rolling back...

//         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
//         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

//         rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient 2>nul
//         rename C:\\inetpub\\wwwroot\\SimpleClient_prev SimpleClient

//         rmdir /S /Q C:\\inetpub\\api\\SimpleAPI 2>nul
//         rename C:\\inetpub\\api\\SimpleAPI_prev SimpleAPI

//         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
//         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

//         exit /b 1

//         :SUCCESS
//         echo Deployment successful
//         exit /b 0
//         '''
//     }
// }



        

// //         stage('Zero-Downtime Deploy with Auto-Rollback') {
// //     steps {
// //         bat '''
// //         setlocal ENABLEDELAYEDEXPANSION

// //         REM ===== STOP IIS =====
// //         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
// //         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

// //         REM ===== PREP =====
// //         rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_new 2>nul
// //         rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_new 2>nul

// //         mkdir C:\\inetpub\\wwwroot\\SimpleClient_new
// //         mkdir C:\\inetpub\\api\\SimpleAPI_new

// //         REM ===== COPY =====
// //         xcopy Angular\\SimpleClient\\dist\\SimpleClient\\browser ^
// //               C:\\inetpub\\wwwroot\\SimpleClient_new /E /I /Y

// //         robocopy out\\SimpleAPI C:\\inetpub\\api\\SimpleAPI_new /MIR /NFL /NDL /NP
// //         IF %ERRORLEVEL% GTR 3 exit /b 1

// //         REM ===== ATOMIC SWAP =====
// //         rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_prev 2>nul
// //         rename C:\\inetpub\\wwwroot\\SimpleClient SimpleClient_prev
// //         rename C:\\inetpub\\wwwroot\\SimpleClient_new SimpleClient

// //         rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_prev 2>nul
// //         rename C:\\inetpub\\api\\SimpleAPI SimpleAPI_prev
// //         rename C:\\inetpub\\api\\SimpleAPI_new SimpleAPI

// //         REM ===== START IIS =====
// //         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
// //         %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool



// //         REM ===== HEALTH CHECK =====
// // ping 127.0.0.1 -n 6 >nul

// // for /f %%s in ('curl -o nul -s -w "%%{http_code}" http://localhost/') do set STATUS=%%s
// // if NOT "%STATUS%"=="200" goto ROLLBACK


// // for /f %%s in ('curl -o nul -s -w "%%{http_code}" http://localhost/api/health') do set API_STATUS=%%s
// // if NOT "%API_STATUS%"=="200" goto ROLLBACK

// // echo Health check OK
// // goto SUCCESS

// // :ROLLBACK
// // echo Health check FAILED. Rolling back...

// // %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
// // %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

// // rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient 2>nul
// // rename C:\\inetpub\\wwwroot\\SimpleClient_prev SimpleClient

// // rmdir /S /Q C:\\inetpub\\api\\SimpleAPI 2>nul
// // rename C:\\inetpub\\api\\SimpleAPI_prev SimpleAPI

// // %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
// // %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool

// // exit /b 1

// // :SUCCESS
// // echo Deployment successful
// // exit /b 0


        

// //         '''
// //     }
// // }




//         // stage('Stop IIS Application Pools') {
//         //     steps {
//         //         bat '''
//         //         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
//         //         %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool
//         //         '''
//         //     }
//         // }

//         // stage('Create backup directories') {
//         //     steps {
//         //         bat '''
//         //         if not exist "%BACKUP_DIR%\\client" mkdir "%BACKUP_DIR%\\client"
//         //         if not exist "%BACKUP_DIR%\\api" mkdir "%BACKUP_DIR%\\api"
//         //         '''
//         //     }
//         // }

//         // stage('Backup existing deployments') {
//         //     steps {
//         //         bat '''
//         //         if exist "C:\\inetpub\\wwwroot\\SimpleClient" (
//         //             xcopy C:\\inetpub\\wwwroot\\SimpleClient "%BACKUP_DIR%\\client" /E /I /Y
//         //         ) else (
//         //             echo Client not deployed yet. Skipping backup.
//         //         )

//         //         if exist "C:\\inetpub\\api\\SimpleAPI" (
//         //             xcopy C:\\inetpub\\api\\SimpleAPI "%BACKUP_DIR%\\api" /E /I /Y
//         //         ) else (
//         //             echo API not deployed yet. Skipping backup.
//         //         )
//         //         '''
//         //     }
//         // }

        

     


//     }

//     post {
//         success {
//             echo '✅ CI/CD pipeline completed successfully'
//         }
//         failure {
//             echo '❌ CI/CD pipeline failed. Investigate logs.'
//         }
//     }
// }
