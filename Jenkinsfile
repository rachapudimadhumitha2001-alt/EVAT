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
                 * Generate JSON security report.
                 * The report is archived after the stage.
                 */
                bat 'npm.cmd audit --json > npm-audit-report.json || exit /b 0'

                echo 'Checking for critical vulnerabilities...'

                /*
                 * Critical vulnerabilities fail the pipeline.
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
                     * Jenkins does not always inherit Docker Desktop's
                     * PATH when running as a Windows service.
                     *
                     * Therefore the absolute Docker path is used.
                     */
                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                    // -------------------------------------------------
                    // Build Docker image
                    // -------------------------------------------------
                    echo 'Building EVAT Docker image...'

                    bat "\"${docker}\" build -t evat-backend:${BUILD_NUMBER} server/node-api"


                    // -------------------------------------------------
                    // Remove previous staging container
                    // -------------------------------------------------
                    echo 'Removing previous staging container if it exists...'

                    /*
                     * Do not use 2>NUL because Jenkins Windows batch
                     * execution previously produced an input-redirection
                     * error.
                     */
                    bat "\"${docker}\" rm -f evat-backend-staging || exit /b 0"


                    // -------------------------------------------------
                    // Deploy staging container
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

                    bat 'powershell -NoProfile -Command "Start-Sleep -Seconds 15"'


                    // -------------------------------------------------
                    // Check deployed container
                    // -------------------------------------------------
                    echo 'Checking deployed container...'

                    bat "\"${docker}\" ps --filter name=evat-backend-staging"


                    // -------------------------------------------------
                    // Check Docker health
                    // -------------------------------------------------
                    echo 'Checking Docker container health...'

                    bat """
                        powershell -NoProfile -Command "\$health=(& '${docker}' inspect --format='{{.State.Health.Status}}' evat-backend-staging); Write-Host ('Docker Health: ' + \$health); if (\$health -eq 'unhealthy') { exit 1 }"
                    """


                    // -------------------------------------------------
                    // Verify API
                    // -------------------------------------------------
                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; Write-Host ('HTTP Status: ' + $response.StatusCode); if ($response.StatusCode -ne 200) { exit 1 }"
                    '''


                    echo 'EVAT deployment completed successfully.'
                }
            }


            // ---------------------------------------------------------
            // Deployment post actions
            // ---------------------------------------------------------
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


                        // -------------------------------------------------
                        // Remove failed staging container
                        // -------------------------------------------------
                        echo 'Removing failed staging container...'

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


        // =========================================================
        // 6. RELEASE
        // =========================================================
        stage('Release') {

            steps {

                script {

                    def docker = 'C:\\Users\\racha\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'


                    // -------------------------------------------------
                    // Create versioned release image
                    // -------------------------------------------------
                    echo 'Creating versioned EVAT release...'

                    bat "\"${docker}\" tag evat-backend:${BUILD_NUMBER} evat-backend:release-${BUILD_NUMBER}"


                    echo "Release image created: evat-backend:release-${BUILD_NUMBER}"


                    // -------------------------------------------------
                    // Capture Git commit
                    // -------------------------------------------------
                    echo 'Capturing Git commit information...'

                    bat 'git rev-parse HEAD > release-commit.txt'


                    // -------------------------------------------------
                    // Create release metadata
                    // -------------------------------------------------
                    echo 'Generating release metadata...'

                    bat """
                        powershell -NoProfile -Command ^
                        "\$commit=(Get-Content release-commit.txt).Trim(); ^
                        \$content=@'
EVAT RELEASE METADATA
=====================
Application: EVAT
Jenkins Build: ${BUILD_NUMBER}
Docker Image: evat-backend:release-${BUILD_NUMBER}
Git Commit: \$commit
Release Status: SUCCESS
'@; ^
                        Set-Content -Path release-metadata.txt -Value \$content"
                    """


                    echo 'EVAT release metadata generated successfully.'
                }
            }


            // ---------------------------------------------------------
            // Release post actions
            // ---------------------------------------------------------
            post {

                success {

                    echo '=========================================='
                    echo 'RELEASE SUCCESSFUL'
                    echo '=========================================='

                    echo "Release image: evat-backend:release-${BUILD_NUMBER}"

                    archiveArtifacts artifacts: 'release-metadata.txt',
                                     allowEmptyArchive: false
                }
            }
        }
    }
}