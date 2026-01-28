pipeline {
    agent any

    tools {
        nodejs 'Node_js'
        jdk 'JDK_HOME'
    }

    environment {
        ANGULAR_DIR = 'Angular/SimpleClient'
        DOTNET_DIR  = 'DotNet/SimpleAPI'
        DIST_DIR    = 'Angular/SimpleClient/dist'
        PUBLISH_DIR = 'DotNet/SimpleAPI/publish'
        BACKUP_DIR = "C:\\inetpub\\backups\\${BUILD_NUMBER}"
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


        //Sonarqube Analysis

        stage('SonarQube Analysis') {
    steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            withSonarQubeEnv('SonarQube-Server') {
                script {
                    def scannerHome = tool 'SonarScanner'
                    bat """
                    "${scannerHome}\\bin\\sonar-scanner.bat" ^
                      -Dsonar.projectKey=Angular-DotNetCICD ^
                      -Dsonar.projectName=Angular-DotNetCICD ^
                      -Dsonar.sources=Angular/SimpleClient/src,DotNet/SimpleAPI ^
                      -Dsonar.exclusions=**/node_modules/**,**/bin/**,**/obj/**,**/publish/** ^
                      -Dsonar.sourceEncoding=UTF-8 ^
                      -Dsonar.token=%SONAR_TOKEN%
                    """
                }
            }
        }
    }
}

        stage('Quality Gate (CE Workaround)') {
    steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            bat '''
            curl -u %SONAR_TOKEN%: ^
            "http://172.27.31.63:9000/api/qualitygates/project_status?projectKey=Angular-DotNetCICD" ^
            -o quality.json

            findstr /C:"ERROR" quality.json && exit /b 1 || echo Quality Gate Passed
            '''
        }
    }
}


        //Quality Gate - Cannot be enforced as I am using Community Edition !!

//         stage('Quality Gate') {
//     steps {
//         timeout(time: 2, unit: 'MINUTES') {
//             waitForQualityGate abortPipeline: true
//         }
//     }
// }

        stage('Archive Artifacts') {
    steps {
        archiveArtifacts artifacts: '''
            Angular/SimpleClient/dist/**,
            out/SimpleAPI/**
        ''', fingerprint: true
    }
}

        stage('Stop IIS Application Pools') {
            steps {
                bat '''
                    %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
                    %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool
                '''
            }
        }

        stage('Create backup directories') {
    steps {
        bat '''
        if not exist "%BACKUP_DIR%\\client" mkdir "%BACKUP_DIR%\\client"
        if not exist "%BACKUP_DIR%\\api" mkdir "%BACKUP_DIR%\\api"
        '''
    }
}
        

        stage('Backup existing deployments') {
            steps {
                bat '''
xcopy C:\\inetpub\\wwwroot\\SimpleClient %BACKUP_DIR%\\client /E /I /Y
        xcopy C:\\inetpub\\api\\SimpleAPI %BACKUP_DIR%\\api /E /I /Y
                '''
            }
        }

        stage('Zero-Downtime Deploy') {
    steps {
        bat '''
        REM === Angular ===
        rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_new 2>nul
        mkdir C:\\inetpub\\wwwroot\\SimpleClient_new

        xcopy Angular\\SimpleClient\\dist\\SimpleClient\\browser ^
              C:\\inetpub\\wwwroot\\SimpleClient_new /E /I /Y

        REM Swap
        rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient_old 2>nul
        rename C:\\inetpub\\wwwroot\\SimpleClient SimpleClient_old
        rename C:\\inetpub\\wwwroot\\SimpleClient_new SimpleClient


        REM === API ===
        rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_new 2>nul
        mkdir C:\\inetpub\\api\\SimpleAPI_new

        xcopy out\\SimpleAPI C:\\inetpub\\api\\SimpleAPI_new /E /I /Y

        rmdir /S /Q C:\\inetpub\\api\\SimpleAPI_old 2>nul
        rename C:\\inetpub\\api\\SimpleAPI SimpleAPI_old
        rename C:\\inetpub\\api\\SimpleAPI_new SimpleAPI
        '''
    }
}

        stage('Start IIS Application Pools') {
            steps {
                bat '''
                %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleClient_AppPool
        %windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:SimpleAPI_AppPool
                '''
            }
        }
    }

    post {
        success {
            echo '✅ CI/CD pipeline completed successfully'
        }
        failure {
            echo '❌ CI/CD pipeline failed. Investigate logs.'
        }
    }
}
