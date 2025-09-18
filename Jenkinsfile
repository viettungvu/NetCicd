			pipeline {
    agent any
    
    environment {
        DOTNET_VERSION = '8.0.x'
        PROJECT_NAME = 'NetCicd'
        SOLUTION_PATH = 'src/NetCicd/NetCicd.csproj'
        PUBLISH_PATH = 'publish'
        DEPLOY_PATH = 'C:\\inetpub\\wwwroot\\NetCicd'
        IIS_SITE_NAME = 'NetCicd'
    }
    
    tools {
        // Cần cài đặt .NET SDK trên Jenkins node
        dotnet 'dotnet-8'
    }
    
    stages {
        stage('Checkout') {
            steps {
                // Checkout code từ GitHub
                checkout scm
                
                // Hiển thị thông tin commit
                script {
                    def commitInfo = sh(returnStdout: true, script: 'git log -1 --pretty=format:"%h - %an: %s"').trim()
                    echo "Building commit: ${commitInfo}"
                }
            }
        }
        
        stage('Restore Dependencies') {
            steps {
                echo 'Restoring NuGet packages...'
                bat 'dotnet restore'
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building the application...'
                bat 'dotnet build --configuration Release --no-restore'
            }
        }
        
        
        stage('Code Quality Analysis') {
            steps {
                echo 'Running code analysis...'
                // SonarQube analysis (nếu có)
                script {
                    if (fileExists('sonar-project.properties')) {
                        bat 'dotnet sonarscanner begin /k:"${PROJECT_NAME}"'
                        bat 'dotnet build'
                        bat 'dotnet sonarscanner end'
                    }
                }
            }
        }
        
        stage('Publish') {
            steps {
                echo 'Publishing the application...'
                bat """
                    dotnet publish "${SOLUTION_PATH}" ^
                    --configuration Release ^
                    --output "${PUBLISH_PATH}" ^
                    --no-build ^
                    --verbosity normal
                """
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                // Lưu trữ artifacts
                archiveArtifacts(
                    artifacts: "${PUBLISH_PATH}/**",
                    allowEmptyArchive: false,
                    fingerprint: true
                )
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Deploying to staging environment...'
                script {
                    deployToEnvironment('staging')
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'
                echo 'Deploying to production environment...'
                script {
                    deployToEnvironment('production')
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            // Cleanup workspace
            cleanWs()
        }
        
        success {
            echo 'Pipeline succeeded!'
            // Gửi thông báo thành công
            script {
                if (env.CHANGE_ID) {
                    // Pull Request
                    pullRequest.comment("✅ Build #${BUILD_NUMBER} succeeded!")
                }
            }
        }
        
        failure {
            echo 'Pipeline failed!'
            // Gửi email thông báo lỗi
            emailext(
                subject: "❌ Build Failed: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: """
                Build failed for ${JOB_NAME} - ${BUILD_NUMBER}
                
                Branch: ${BRANCH_NAME}
                Commit: ${GIT_COMMIT}
                
                Check console output: ${BUILD_URL}
                """,
                to: "${env.CHANGE_AUTHOR_EMAIL ?: 'admin@company.com'}"
            )
        }
    }
}

// Function để deploy tới các environment khác nhau
def deployToEnvironment(environment) {
    echo "Deploying to ${environment} environment..."
    
    // Stop IIS site
    bat """
        powershell -Command "Import-Module WebAdministration; Stop-WebSite -Name '${IIS_SITE_NAME}' -ErrorAction SilentlyContinue"
    """
    
    // Backup current version
    bat """
        powershell -Command "
            \\$backupPath = '${DEPLOY_PATH}_backup_' + (Get-Date -Format 'yyyyMMdd_HHmmss')
            if (Test-Path '${DEPLOY_PATH}') {
                Copy-Item -Path '${DEPLOY_PATH}' -Destination \\$backupPath -Recurse
                Write-Host 'Backup created at: ' + \\$backupPath
            }
        "
    """
    
    // Deploy new version
    bat """
        powershell -Command "
            if (Test-Path '${DEPLOY_PATH}') {
                Remove-Item -Path '${DEPLOY_PATH}\\*' -Recurse -Force -Exclude 'Logs'
            } else {
                New-Item -ItemType Directory -Path '${DEPLOY_PATH}' -Force
            }
            Copy-Item -Path '${PUBLISH_PATH}\\*' -Destination '${DEPLOY_PATH}' -Recurse -Force
        "
    """
    
    // Start IIS site
    bat """
        powershell -Command "Import-Module WebAdministration; Start-WebSite -Name '${IIS_SITE_NAME}'"
    """
    
    // Health check
    sleep(10)
    script {
        def response = bat(returnStatus: true, script: 'curl -f http://localhost/health')
        if (response != 0) {
            error("Health check failed!")
        }
    }
    
    echo "Deployment to ${environment} completed successfully!"
}