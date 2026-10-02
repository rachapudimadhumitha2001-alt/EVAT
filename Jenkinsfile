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
            steps {
                script {

                    // Jenkins cannot currently find Docker through PATH,
                    // so use the verified Docker executable directly.
                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                    // ------------------------------------------------
                    // Build Docker image
                    // ------------------------------------------------
                    echo 'Building Docker image...'

                    bat "\"${docker}\" build -t evat-backend:${BUILD_NUMBER} server/node-api"


                    // ------------------------------------------------
                    // Remove previous staging container
                    // ------------------------------------------------
                    echo 'Removing previous staging container if it exists...'

                    bat "\"${docker}\" rm -f evat-backend-staging 2>NUL || exit /b 0"


                    // ------------------------------------------------
                    // Start new staging container
                    // ------------------------------------------------
                    echo 'Deploying EVAT backend container...'

                    bat "\"${docker}\" run -d --name evat-backend-staging -p 8082:8081 --env-file server/node-api/.env -e MONGODB_URI=mongodb://host.docker.internal:27017/evat evat-backend:${BUILD_NUMBER}"


                    // ------------------------------------------------
                    // Give application time to start
                    // ------------------------------------------------
                    echo 'Waiting for EVAT container to start...'

                    bat 'timeout /t 10 /nobreak'


                    // ------------------------------------------------
                    // Check container status
                    // ------------------------------------------------
                    echo 'Checking deployed container...'

                    bat "\"${docker}\" ps --filter name=evat-backend-staging"


                    // ------------------------------------------------
                    // API health check
                    // ------------------------------------------------
                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; Write-Host ('HTTP Status: ' + $response.StatusCode); if ($response.StatusCode -ne 200) { exit 1 }"
                    '''


                    // ------------------------------------------------
                    // Deployment successful
                    // ------------------------------------------------
                    echo 'EVAT deployment completed successfully.'
                }
            }

            post {

                // ----------------------------------------------------
                // Rollback if deployment fails
                // ----------------------------------------------------
                failure {
                    echo 'Deployment failed. Attempting rollback...'

                    script {

                        def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                        // Remove failed staging container
                        bat "\"${docker}\" rm -f evat-backend-staging 2>NUL || exit /b 0"


                        // Restore known-good version
                        bat "\"${docker}\" run -d --name evat-backend -p 8082:8081 --env-file server/node-api/.env -e MONGODB_URI=mongodb://host.docker.internal:27017/evat evat-backend:1.1"


                        echo 'Rollback attempted using known-good evat-backend:1.1.'
                    }
                }


                // ----------------------------------------------------
                // Deployment successful
                // ----------------------------------------------------
                success {
                    echo 'Deployment and health checks passed.'
                }
            }
        }
    }
}