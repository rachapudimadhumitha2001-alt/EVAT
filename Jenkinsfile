pipeline {
    agent any

    stages {

        // ============================================================
        // 1. BUILD
        // ============================================================
        stage('Build') {
            steps {
                echo 'Building EVAT application...'

                bat 'npm ci'
                bat 'npm run build:client'
                bat 'npm run build:server'
            }
        }


        // ============================================================
        // 2. TEST
        // ============================================================
        stage('Test') {
            steps {
                echo 'Running EVAT automated tests...'

                bat 'npm run test:server -- --runInBand'
            }
        }


        // ============================================================
        // 3. CODE QUALITY
        // ============================================================
        stage('Code Quality') {
            steps {
                echo 'Running SonarQube code quality analysis...'

                withSonarQubeEnv('EVAT-SonarQube') {
                    script {
                        def scannerHome = tool 'SonarQube-Scanner'

                        bat "\"${scannerHome}\\bin\\sonar-scanner.bat\""
                    }
                }
            }
        }


        // ============================================================
        // 4. SECURITY
        // ============================================================
        stage('Security') {
            steps {
                echo 'Running npm dependency security audit...'

                bat 'npm.cmd audit --json > npm-audit-report.json || exit /b 0'

                echo 'Checking for critical vulnerabilities...'

                bat 'npm.cmd audit --audit-level=critical'
            }

            post {
                always {
                    archiveArtifacts artifacts: 'npm-audit-report.json',
                                     allowEmptyArchive: true
                }
            }
        }


        // ============================================================
        // 5. DEPLOY
        // ============================================================
        stage('Deploy') {

            environment {
                EVAT_JWT_SECRET = credentials('evat-jwt-secret')
            }

            steps {
                script {

                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

                    echo 'Building Docker image...'

                    bat "\"${docker}\" build -t evat-backend:${BUILD_NUMBER} server/node-api"


                    echo 'Removing previous staging container if it exists...'

                    bat "\"${docker}\" rm -f evat-backend-staging 2>NUL || exit /b 0"


                    echo 'Deploying EVAT backend container...'

                    bat """
                        "${docker}" run -d ^
                        --name evat-backend-staging ^
                        -p 8082:8081 ^
                        -e "DOMAIN_URL=" ^
                        -e "CLIENT_ORIGIN=http://localhost:3000" ^
                        -e "COOKIE_SAME_SITE=lax" ^
                        -e "MONGODB_URI=mongodb://host.docker.internal:27017/evat" ^
                        -e "JWT_SECRET=%EVAT_JWT_SECRET%" ^
                        -e "PORT=8081" ^
                        -e "PYTHON_API_URL=http://host.docker.internal:5000" ^
                        -e "COST_API_URL=" ^
                        -e "DEMAND_API_URL=" ^
                        -e "RELIABILITY_API_URL=http://host.docker.internal:8003" ^
                        evat-backend:${BUILD_NUMBER}
                    """


                    echo 'Waiting for Docker health check...'

                    bat 'timeout /t 10 /nobreak'


                    echo 'Checking deployed container...'

                    bat "\"${docker}\" ps --filter name=evat-backend-staging"


                    echo 'Checking Docker container health...'

                    bat """
                        powershell -NoProfile -Command "\$health=(docker inspect --format='{{.State.Health.Status}}' evat-backend-staging); Write-Host ('Docker Health: ' + \$health); if (\$health -eq 'unhealthy') { exit 1 }"
                    """


                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; Write-Host ('HTTP Status: ' + $response.StatusCode); if ($response.StatusCode -ne 200) { exit 1 }"
                    '''


                    echo 'EVAT deployment completed successfully.'
                }
            }

            post {

                // --------------------------------------------------------
                // Rollback
                // --------------------------------------------------------
                failure {

                    echo 'Deployment failed. Attempting rollback...'

                    script {

                        def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                        echo 'Removing failed staging container...'

                        bat "\"${docker}\" rm -f evat-backend-staging 2>NUL || exit /b 0"


                        echo 'Starting rollback using known-good image evat-backend:1.1...'

                        bat """
                            "${docker}" run -d ^
                            --name evat-backend ^
                            -p 8082:8081 ^
                            -e "DOMAIN_URL=" ^
                            -e "CLIENT_ORIGIN=http://localhost:3000" ^
                            -e "COOKIE_SAME_SITE=lax" ^
                            -e "MONGODB_URI=mongodb://host.docker.internal:27017/evat" ^
                            -e "JWT_SECRET=%EVAT_JWT_SECRET%" ^
                            -e "PORT=8081" ^
                            evat-backend:1.1
                        """


                        echo 'Rollback attempted using known-good evat-backend:1.1.'
                    }
                }


                // --------------------------------------------------------
                // Successful deployment
                // --------------------------------------------------------
                success {

                    echo 'Deployment and health checks passed.'
                }
            }
        }
    }
}