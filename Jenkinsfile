pipeline {
    agent any
    environment {
        DOTNET_VERSION = '8.0.x' // Phiên bản .NET SDK
        BRANCH='master'
        GIT_URL = 'https://github.com/viettungvu/NetCicd.git' // Đường dẫn project file
        SOLUTION_PATH = 'NetCicd.csproj' // Đường dẫn project file
        PUBLISH_DIR = 'publish' // Folder output publish
        DEPLOY_SERVER = '123.30.238.37'
        DEPLOY_DIR = 'E:\\AppTools\\DemoJenkins'
        SERVICE_NAME='XM.ATS.DemoJenkins'
		SERVICE_EXCUTABLE='NetCicd.exe'
        ZIP_FILE = 'publish.zip'
        CREDENTIAL_ID='deploy-credential-37'
        // Di chuyển BACKUP_ZIP vào script PowerShell để generate timestamp động
    }
    stages {
        stage('Checkout Source Code') {
            steps {
                script {
                    try {
                        checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/viettungvu/NetCicd.git']]) // Checkout từ GitHub
                        echo "Checkout thành công từ ${env.GIT_URL}"
                    } catch (Exception e) {
                        error "Checkout thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Checkout hoàn tất.'
                }
                failure {
                    echo 'Bước Checkout thất bại. Dừng pipeline.'
                }
            }
        }
        stage('Restore Dependencies') {
            steps {
                script {
                    try {
                        bat "dotnet restore ${SOLUTION_PATH}" // Hoặc sh nếu Linux
                        echo "Restore dependencies thành công."
                    } catch (Exception e) {
                        error "Restore thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Restore hoàn tất.'
                }
                failure {
                    echo 'Bước Restore thất bại. Dừng pipeline.'
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    try {
                        bat "dotnet build ${SOLUTION_PATH} --configuration Release --no-restore"
                        echo "Build thành công."
                    } catch (Exception e) {
                        error "Build thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Build hoàn tất.'
                }
                failure {
                    echo 'Bước Build thất bại. Dừng pipeline.'
                }
            }
        }
        
        stage('Publish') {
            steps {
                script {
                    try {
                        bat "dotnet publish ${SOLUTION_PATH} --configuration Release --output ${PUBLISH_DIR} --no-build"
                        // Tạo file ZIP từ thư mục publish
                        bat """
                            powershell Compress-Archive -Path '${PUBLISH_DIR}\\*' -DestinationPath '${ZIP_FILE}' -Force
                            echo "Đã tạo file ZIP: ${ZIP_FILE}"
                        """
                        echo "Publish thành công vào folder ${PUBLISH_DIR}."
                    } catch (Exception e) {
                        error "Publish thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Publish hoàn tất. Archive artifacts.'
                    archiveArtifacts artifacts: "${ZIP_FILE}", allowEmptyArchive: false
                }
                failure {
                    echo 'Bước Publish thất bại. Dừng pipeline.'
                }
            }
        }
        stage('Deploy to IIS') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'deploy-credential-37', usernameVariable: 'DEPLOY_USER', passwordVariable: 'DEPLOY_PASS')]) {
                            powershell """
                                \$securePass = ConvertTo-SecureString \$env:DEPLOY_PASS -AsPlainText -Force
                                \$cred = New-Object System.Management.Automation.PSCredential (\$env:DEPLOY_USER, \$securePass)
                                \$session = New-PSSession -ComputerName ${DEPLOY_SERVER} -Credential \$cred -Authentication Basic # Hoặc Kerberos nếu domain
                                \$backupZip = "backup-\$(Get-Date -Format 'yyyyMMdd-HHmmss').zip"
                                # Kiểm tra và tạo thư mục DEPLOY_DIR nếu không tồn tại
                                Invoke-Command -Session \$session -ScriptBlock {
                                    if (!(Test-Path '${DEPLOY_DIR}')) {
                                        New-Item -ItemType Directory -Path '${DEPLOY_DIR}' -Force | Out-Null
                                        Write-Output 'Đã tạo thư mục ${DEPLOY_DIR}'
                                    }
                                }
                                
                                # Copy zip sang deploy server
                                Copy-Item -ToSession \$session -Path "${ZIP_FILE}" -Destination "${DEPLOY_DIR}\\${ZIP_FILE}" -Force
                                Write-Output 'Đã copy file ZIP sang server'
                                
                                # Kiểm tra service tồn tại trước khi stop (nếu không tồn tại thì skip)
                                \$serviceExists = Invoke-Command -Session \$session -ScriptBlock { 
                                    \$svc = Get-Service -Name '${SERVICE_NAME}' -ErrorAction SilentlyContinue
                                    return \$null -ne \$svc
                                }
                                
                                if (\$serviceExists) {
                                    Invoke-Command -Session \$session -ScriptBlock { 
                                        Stop-Service -Name '${SERVICE_NAME}' -Force -ErrorAction SilentlyContinue 
                                        Write-Output 'Đã stop service ${SERVICE_NAME}'
                                    }
                                } else {
									Write-Output 'Service ${SERVICE_NAME} không tồn tại, sẽ tạo mới'
									# Tạo service mới
									Invoke-Command -Session $session -ScriptBlock { 
										$exePath = Join-Path '${DEPLOY_DIR}' '{SERVICE_EXCUTABLE}'
										if (Test-Path $exePath) {
											New-Service -Name '${SERVICE_NAME}' -BinaryPathName $exePath -StartupType Automatic -DisplayName '${SERVICE_NAME}' | Out-Null
											Write-Output "Đã tạo service ${SERVICE_NAME} từ $exePath"
										} else {
											throw "File exe không tồn tại: $exePath"
										}
									}
                                }
                                
                                # Giải nén zip mới
                                Invoke-Command -Session \$session -ScriptBlock { 
                                    Expand-Archive -Path '${DEPLOY_DIR}\\${ZIP_FILE}' -DestinationPath '${DEPLOY_DIR}' -Force 
                                    Write-Output 'Đã giải nén file ZIP mới'
                                }
                                # Start service
								Invoke-Command -Session \$session -ScriptBlock { 
									Start-Service -Name '${SERVICE_NAME}' -ErrorAction SilentlyContinue
									Write-Output 'Đã start service ${SERVICE_NAME}'
								}
                                
                                # Kiểm tra service running và xóa zip nếu thành công
                                \$status = Invoke-Command -Session \$session -ScriptBlock { (Get-Service -Name '${SERVICE_NAME}').Status }
                                if (\$status -eq 'Running') {
                                    Invoke-Command -Session \$session -ScriptBlock { 
                                        Remove-Item -Path '${DEPLOY_DIR}\\${ZIP_FILE}' -Force -ErrorAction SilentlyContinue 
                                    }
                                    Write-Output "Deploy thành công, xóa zip."
                                } else {
                                    throw "Service không start được! Status: \$status"
                                }
                                
                                Remove-PSSession \$session
                            """
                        }
                    } catch (Exception e) {
                        error "Deploy thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Deploy hoàn tất. Ứng dụng sẵn sàng trên IIS.'
                    // Xóa thư mục publish và ZIP sau khi deploy thành công
                    bat """
                        if exist "${PUBLISH_DIR}" rmdir /s /q "${PUBLISH_DIR}"
                        if exist "${ZIP_FILE}" del "${ZIP_FILE}"
                        echo "Đã xóa thư mục ${PUBLISH_DIR} và file ${ZIP_FILE}."
                    """
                }
                failure {
                    echo 'Bước Deploy thất bại. Kiểm tra logs để biết chi tiết.'
                    // Không xóa thư mục nếu deploy thất bại để debug
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline kết thúc. Kiểm tra logs để xem chi tiết.'
            // Cleanup nếu cần: bat "rmdir /s /q ${PUBLISH_DIR}"
        }
        success {
            echo 'Toàn bộ pipeline thành công!'
            // Gửi email/slack notification nếu cần
        }
        failure {
            echo 'Pipeline thất bại. Kiểm tra bước lỗi.'
            // Gửi alert
        }
    }
}