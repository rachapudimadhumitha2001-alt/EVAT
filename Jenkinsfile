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

                /*
                 * Generate a complete JSON security report.
                 *
                 * The || exit /b 0 allows the report to be archived
                 * even when npm audit finds vulnerabilities.
                 */
                bat 'npm.cmd audit --json > npm-audit-report.json || exit /b 0'

                echo 'Checking for critical vulnerabilities...'

                /*
                 * Critical vulnerabilities fail the pipeline.
                 * Lower severity vulnerabilities are reported but do
                 * not currently block deployment.
                 */
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

                    /*
                     * Jenkins running as a Windows service does not
                     * reliably inherit Docker Desktop's PATH.
                     *
                     * Therefore we explicitly use the Docker executable.
                     */
                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                    // -------------------------------------------------
                    // Build Docker image
                    // -------------------------------------------------
                    echo 'Building EVAT Docker image...'

                    bat "\"${docker}\" build -t evat-backend:${BUILD_NUMBER} server/node-api"


                    // -------------------------------------------------
                    // Remove old staging container
                    // -------------------------------------------------
                    echo 'Removing previous staging container if it exists...'

                    /*
                     * Do not use 2>NUL here.
                     *
                     * Jenkins on Windows was producing:
                     *
                     * ERROR: Input redirection is not supported
                     *
                     * The command is allowed to fail if the container
                     * does not exist.
                     */
                    bat "\"${docker}\" rm -f evat-backend-staging || exit /b 0"


                    // -------------------------------------------------
                    // Start new staging container
                    // -------------------------------------------------
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


                    // -------------------------------------------------
                    // Wait for application startup
                    // -------------------------------------------------
                    echo 'Waiting for Docker application to start...'

                    /*
                     * PowerShell Start-Sleep is more reliable than
                     * Windows timeout when Jenkins is running as a
                     * non-interactive service.
                     */
                    bat 'powershell -NoProfile -Command "Start-Sleep -Seconds 15"'


                    // -------------------------------------------------
                    // Check container is running
                    // -------------------------------------------------
                    echo 'Checking deployed container...'

                    bat "\"${docker}\" ps --filter name=evat-backend-staging"


                    // -------------------------------------------------
                    // Check Docker health status
                    // -------------------------------------------------
                    echo 'Checking Docker container health...'

                    bat """
                        powershell -NoProfile -Command "\$health=(& '${docker}' inspect --format='{{.State.Health.Status}}' evat-backend-staging); Write-Host ('Docker Health: ' + \$health); if (\$health -eq 'unhealthy') { exit 1 }"
                    """


                    // -------------------------------------------------
                    // Verify API endpoint
                    // -------------------------------------------------
                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; Write-Host ('HTTP Status: ' + $response.StatusCode); if ($response.StatusCode -ne 200) { exit 1 }"
                    '''


                    // -------------------------------------------------
                    // Deployment completed
                    // -------------------------------------------------
                    echo 'EVAT deployment completed successfully.'
                }
            }


            // =========================================================
            // DEPLOY POST ACTIONS
            // =========================================================
            post {

                // -----------------------------------------------------
                // Deployment SUCCESS
                // -----------------------------------------------------
                success {

                    echo '=========================================='
                    echo 'DEPLOYMENT SUCCESSFUL'
                    echo '=========================================='

                    echo 'Docker container is running.'
                    echo 'Docker health check passed.'
                    echo 'EVAT API endpoint verification passed.'
                    echo 'Staging deployment completed successfully.'
                }


                // -----------------------------------------------------
                // Deployment FAILURE
                // -----------------------------------------------------
                failure {

                    echo '=========================================='
                    echo 'DEPLOYMENT FAILED'
                    echo 'ATTEMPTING ROLLBACK'
                    echo '=========================================='

                    script {

                        def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                        // -------------------------------------------------
                        // Remove failed staging container
                        // -------------------------------------------------
                        echo 'Removing failed staging container...'

                        /*
                         * No 2>NUL redirection.
                         */
                        bat "\"${docker}\" rm -f evat-backend-staging || exit /b 0"


                        // -------------------------------------------------
                        // Rollback
                        // -------------------------------------------------
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
    }
}