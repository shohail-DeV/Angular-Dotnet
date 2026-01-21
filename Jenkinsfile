pipeline {
    agent any

    tools {
        nodejs 'Node_js'
    }

    environment {
        ANGULAR_DIR = 'Angular/SimpleClient'
        DOTNET_DIR  = 'DotNet/SimpleAPI'
        DIST_DIR    = 'Angular/SimpleClient/dist'
        PUBLISH_DIR = 'DotNet/SimpleAPI/publish'
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
                    bat 'dotnet publish SimpleAPI.csproj -c Release -o publish'
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '''
                    Angular/SimpleClient/dist/**,
                    DotNet/SimpleAPI/publish/**
                ''', fingerprint: true
            }
        }
        
        stage('Deploy to IIS') {
    steps {
        bat '''

        REM Set backup directory with build number
        set BACKUP_DIR=C:\\inetpub\\backups\\%BUILD_NUMBER%

        REM Stop IIS application pools
        %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleClient_AppPool
        %windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:SimpleAPI_AppPool

        REM Create backup directories
        mkdir %BACKUP_DIR%\\client
        mkdir %BACKUP_DIR%\\api

        REM Backup existing deployments
        xcopy C:\\inetpub\\wwwroot\\SimpleClient %BACKUP_DIR%\\client /E /I /Y
        xcopy C:\\inetpub\\api\\SimpleAPI %BACKUP_DIR%\\api /E /I /Y

        REM Deploy new builds
        rmdir /S /Q C:\\inetpub\\wwwroot\\SimpleClient
        mkdir C:\\inetpub\\wwwroot\\SimpleClient
        xcopy Angular\\SimpleClient\\dist\\SimpleClient\\browser C:\\inetpub\\wwwroot\\SimpleClient /E /I /Y

        REM Deploy .NET API
        rmdir /S /Q C:\\inetpub\\api\\SimpleAPI
        mkdir C:\\inetpub\\api\\SimpleAPI
        xcopy DotNet\\SimpleAPI\\publish C:\\inetpub\\api\\SimpleAPI /E /I /Y

        REM Start IIS application pools
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
