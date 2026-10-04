pipeline {
    agent any

    stages {

        // =========================================================
        // 1. BUILD
        // =========================================================
        stage('Build') {
            steps {
                echo 'Building EVAT application...'

                bat 'npm ci'

                echo 'Building React frontend...'
                bat 'npm run build:client'

                echo 'Building Node.js/TypeScript backend...'
                bat 'npm run build:server'

                echo 'Build stage completed successfully.'
            }
        }

        // =========================================================
        // 2. TEST
        // =========================================================
        stage('Test') {
            steps {
                echo 'Running EVAT automated tests...'

                bat 'npm run test:server -- --runInBand'

                echo 'Test stage completed successfully.'
            }
        }

        // =========================================================
        // 3. CODE QUALITY
        // =========================================================
        stage('Code Quality') {
            steps {
                echo 'Running SonarQube code quality analysis...'

                withSonarQubeEnv('EVAT-SonarQube') {
                    script {
                        def scannerHome = tool 'SonarQube-Scanner'

                        bat "\"${scannerHome}\\bin\\sonar-scanner.bat\""
                    }
                }

                echo 'Code quality analysis completed successfully.'
            }
        }

        // =========================================================
        // 4. SECURITY
        // =========================================================
        stage('Security') {
            steps {
                echo 'Running npm dependency security audit...'

                bat 'npm.cmd audit --json > npm-audit-report.json || exit /b 0'

                echo 'Checking for critical vulnerabilities...'

                bat 'npm.cmd audit --audit-level=critical'

                echo 'Security stage completed successfully.'
            }

            post {
                always {
                    echo 'Archiving npm security audit report...'

                    archiveArtifacts artifacts: 'npm-audit-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // =========================================================
        // 5. DEPLOY
        // =========================================================
        stage('Deploy') {

            environment {
                EVAT_JWT_SECRET = credentials('evat-jwt-secret')
            }

            steps {
                script {

                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

                    echo 'Building EVAT Docker image...'

                    bat "\"${docker}\" build -t evat-backend:${BUILD_NUMBER} server/node-api"

                    echo 'Removing previous staging container if it exists...'

                    bat "\"${docker}\" rm -f evat-backend-staging || exit /b 0"

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

                    echo 'Waiting for Docker application to start...'

                    bat 'powershell -NoProfile -Command "Start-Sleep -Seconds 15"'

                    echo 'Checking deployed container...'

                    bat "\"${docker}\" ps --filter name=evat-backend-staging"

                    echo 'Checking Docker container health...'

                    bat """
                        powershell -NoProfile -Command "\$health=(& '${docker}' inspect --format='{{.State.Health.Status}}' evat-backend-staging); Write-Host ('Docker Health: ' + \$health); if (\$health -eq 'unhealthy') { exit 1 }"
                    """

                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; Write-Host ('HTTP Status: ' + $response.StatusCode); if ($response.StatusCode -ne 200) { exit 1 }"
                    '''

                    echo 'EVAT deployment completed successfully.'
                }
            }

            post {

                success {
                    echo '=========================================='
                    echo 'DEPLOYMENT SUCCESSFUL'
                    echo '=========================================='
                    echo 'Docker container is running.'
                    echo 'Docker health check passed.'
                    echo 'EVAT API endpoint verification passed.'
                    echo 'Staging deployment completed successfully.'
                }

                failure {
                    echo '=========================================='
                    echo 'DEPLOYMENT FAILED'
                    echo 'ATTEMPTING ROLLBACK'
                    echo '=========================================='

                    script {

                        def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

                        echo 'Removing failed staging container...'

                        bat "\"${docker}\" rm -f evat-backend-staging || exit /b 0"

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
            }
        }

        // =========================================================
        // 6. RELEASE
        // =========================================================
        stage('Release') {

            steps {
                script {

                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

                    echo '=========================================='
                    echo 'CREATING VERSIONED EVAT RELEASE'
                    echo '=========================================='

                    // Promote the exact Docker image that passed staging.
                    bat "\"${docker}\" tag evat-backend:${BUILD_NUMBER} evat-backend:release-${BUILD_NUMBER}"

                    echo "Release image created: evat-backend:release-${BUILD_NUMBER}"

                    // Capture Git commit.
                    bat 'git rev-parse HEAD > release-commit.txt'

                    // Create simple release metadata without PowerShell
                    // here-strings, avoiding Windows batch escaping issues.
                    bat 'echo EVAT RELEASE METADATA> release-metadata.txt'
                    bat 'echo Application: EVAT>> release-metadata.txt'
                    bat 'echo Jenkins Build: %BUILD_NUMBER%>> release-metadata.txt'
                    bat 'echo Docker Image: evat-backend:release-%BUILD_NUMBER%>> release-metadata.txt'
                    bat 'echo Git Commit:>> release-metadata.txt'
                    bat 'type release-commit.txt >> release-metadata.txt'
                    bat 'echo Release Status: SUCCESS>> release-metadata.txt'

                    echo 'Release metadata generated successfully.'

                    echo 'Release image verification...'

                    bat "\"${docker}\" image inspect evat-backend:release-${BUILD_NUMBER}"

                    echo '=========================================='
                    echo "RELEASE SUCCESSFUL: evat-backend:release-${BUILD_NUMBER}"
                    echo '=========================================='
                }
            }

            post {
                success {

                    echo 'Archiving release metadata...'

                    archiveArtifacts artifacts: 'release-metadata.txt,release-commit.txt',
                                     allowEmptyArchive: false

                    echo 'Versioned release artefact archived successfully.'
                }

                failure {
                    echo '=========================================='
                    echo 'RELEASE FAILED'
                    echo '=========================================='
                }
            }
        }

        // =========================================================
        // 7. MONITORING
        // =========================================================
        stage('Monitoring') {

            steps {
                script {

                    echo '=========================================='
                    echo 'EVAT MONITORING CHECK'
                    echo '=========================================='

                    // -------------------------------------------------
                    // 7.1 Check cAdvisor availability through Prometheus
                    // -------------------------------------------------
                    echo 'Checking Prometheus cAdvisor target...'

                    def healthResponse = bat(
                        returnStdout: true,
                        script: '@curl.exe -s "http://127.0.0.1:9090/api/v1/query?query=up%7Bjob%3D%22cadvisor%22%7D"'
                    ).trim()

                    echo 'Prometheus health response:'
                    echo healthResponse

                    if (!healthResponse.contains('"status":"success"') ||
                        !healthResponse.contains('"value"') ||
                        !healthResponse.contains('"1"')) {

                        error 'MONITORING FAILED: cAdvisor target is not UP in Prometheus.'
                    }

                    echo 'cAdvisor monitoring target: UP'


                    // -------------------------------------------------
                    // 7.2 Check EVAT staging CPU metric
                    // -------------------------------------------------
                    echo 'Checking EVAT staging CPU metric...'

                    def cpuResponse = bat(
                        returnStdout: true,
                        script: '@curl.exe -s "http://127.0.0.1:9090/api/v1/query?query=container_cpu_usage_seconds_total%7Bname%3D%22evat-backend-staging%22%7D"'
                    ).trim()

                    echo 'EVAT CPU monitoring response:'
                    echo cpuResponse

                    if (!cpuResponse.contains('"status":"success"') ||
                        !cpuResponse.contains('evat-backend-staging')) {

                        error 'MONITORING FAILED: EVAT staging CPU metric not found.'
                    }

                    echo 'EVAT CPU metric: AVAILABLE'


                    // -------------------------------------------------
                    // 7.3 Check EVAT staging memory metric
                    // -------------------------------------------------
                    echo 'Checking EVAT staging memory metric...'

                    def memoryResponse = bat(
                        returnStdout: true,
                        script: '@curl.exe -s "http://127.0.0.1:9090/api/v1/query?query=container_memory_usage_bytes%7Bname%3D%22evat-backend-staging%22%7D"'
                    ).trim()

                    echo 'EVAT memory monitoring response:'
                    echo memoryResponse

                    if (!memoryResponse.contains('"status":"success"') ||
                        !memoryResponse.contains('evat-backend-staging')) {

                        error 'MONITORING FAILED: EVAT staging memory metric not found.'
                    }

                    echo 'EVAT memory metric: AVAILABLE'


                    // -------------------------------------------------
                    // 7.4 Final monitoring result
                    // -------------------------------------------------
                    echo '=========================================='
                    echo 'MONITORING CHECK PASSED'
                    echo '=========================================='
                    echo 'Prometheus target: UP'
                    echo 'EVAT CPU metric: AVAILABLE'
                    echo 'EVAT memory metric: AVAILABLE'
                    echo '=========================================='
                }
            }

            post {

                success {
                    echo 'Monitoring verification completed successfully.'
                }

                failure {
                    echo '=========================================='
                    echo 'MONITORING FAILED'
                    echo '=========================================='
                }
            }
        }
    }
}
