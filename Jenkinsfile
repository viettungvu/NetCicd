pipeline {
    agent any

    environment {
        DOTNET_VERSION = '8.0.x'  // Phiên bản .NET SDK
        SOLUTION_PATH = 'NetCicd.sln'  // Đường dẫn solution file
        PUBLISH_DIR = 'publish'  // Folder output publish
        IIS_SITE_NAME = 'jenkins-demo'  // Tên IIS site
		IIS_APP_POOL = 'jenkins-demo'
        IIS_SERVER = 'localhost'  // IP/hostname IIS server (thay bằng remote nếu cần)
		IIS_DEPLOY_PATH = 'D:\\Jenkins\\demo'
        // Thêm credentials nếu remote: withCredentials([usernamePassword(credentialsId: 'iis-creds', ...)])
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                script {
                    try {
                        checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/viettungvu/NetCicd.git']])  // Checkout từ GitHub
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
                        bat "dotnet restore ${SOLUTION_PATH}"  // Hoặc sh nếu Linux
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

        stage('Test') {  // Optional, thêm nếu có unit tests
            steps {
                script {
                    try {
                        bat "dotnet test ${SOLUTION_PATH} --configuration Release --no-build --logger trx"
                        echo "Test thành công."
                    } catch (Exception e) {
                        error "Test thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Test hoàn tất.'
                }
                failure {
                    echo 'Bước Test thất bại. Dừng pipeline.'  // Hoặc unstable nếu muốn tiếp tục
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    try {
                        bat "dotnet publish ${SOLUTION_PATH} --configuration Release --output ${PUBLISH_DIR} --no-build"
                        echo "Publish thành công vào folder ${PUBLISH_DIR}."
                    } catch (Exception e) {
                        error "Publish thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Publish hoàn tất. Archive artifacts.'
                    archiveArtifacts artifacts: "${PUBLISH_DIR}/**", allowEmptyArchive: false
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
                        // Copy publish folder đến thư mục mới
                        bat """
                            xcopy /Y /E /I ${PUBLISH_DIR} "${IIS_DEPLOY_PATH}\\"
                        """

                        // PowerShell để kiểm tra và restart AppPool/Site
                        powershell """
                            Import-Module WebAdministration -ErrorAction Stop

                            # Kiểm tra thư mục deploy tồn tại
                            if (-not (Test-Path '${IIS_DEPLOY_PATH}')) {
                                Write-Error 'Thư mục ${IIS_DEPLOY_PATH} không tồn tại!'
                                exit 1
                            }

                            # Kiểm tra AppPool tồn tại
                            if (Get-WebAppPoolState -Name '${IIS_APP_POOL}' -ErrorAction SilentlyContinue) {
                                Write-Output 'AppPool ${IIS_APP_POOL} tồn tại. Restarting...'
                                Restart-WebAppPool -Name '${IIS_APP_POOL}' -ErrorAction Stop
                            } else {
                                Write-Error 'AppPool ${IIS_APP_POOL} không tồn tại!'
                                exit 1
                            }

                            # Kiểm tra Site tồn tại
                            if (Get-Website -Name '${IIS_SITE_NAME}' -ErrorAction SilentlyContinue) {
                                Write-Output 'Site ${IIS_SITE_NAME} tồn tại. Restarting...'
                                Stop-Website -Name '${IIS_SITE_NAME}' -ErrorAction Stop
                                Start-Website -Name '${IIS_SITE_NAME}' -ErrorAction Stop
                            } else {
                                Write-Error 'Site ${IIS_SITE_NAME} không tồn tại!'
                                exit 1
                            }

                            Write-Output 'Deploy và restart IIS thành công.'
                        """
                        echo "Deploy lên IIS thành công."
                    } catch (Exception e) {
                        error "Deploy thất bại: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo 'Bước Deploy hoàn tất. Ứng dụng sẵn sàng trên IIS.'
                }
                failure {
                    echo 'Bước Deploy thất bại. Kiểm tra logs để biết chi tiết.'
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