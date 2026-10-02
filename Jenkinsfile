pipeline {
    agent any

    stages {

        stage('Build') {
            steps {
                echo 'Building EVAT application...'

                bat 'npm ci'
                bat 'npm run build:client'
                bat 'npm run build:server'
            }
        }

        stage('Test') {
            steps {
                echo 'Running EVAT automated tests...'

                bat 'npm run test:server -- --runInBand'
            }
        }

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

        stage('Deploy') {
            steps {
                script {

                    echo 'Building Docker image...'

                    bat "docker build -t evat-backend:${BUILD_NUMBER} server/node-api"

                    echo 'Deploying EVAT backend container...'

                    bat 'docker rm -f evat-backend-staging 2>NUL || exit /b 0'

                    bat "docker run -d --name evat-backend-staging -p 8082:8081 --env-file server/node-api/.env -e MONGODB_URI=mongodb://host.docker.internal:27017/evat evat-backend:${BUILD_NUMBER}"

                    echo 'Waiting for Docker health check...'

                    bat '''
                        powershell -NoProfile -Command ^
                        "$deadline=(Get-Date).AddMinutes(2); ^
                        do { ^
                            $status=docker inspect --format="{{.State.Health.Status}}" evat-backend-staging 2>$null; ^
                            Write-Host "Container health: $status"; ^
                            if ($status -eq "healthy") { exit 0 }; ^
                            if ($status -eq "unhealthy") { exit 1 }; ^
                            Start-Sleep -Seconds 5 ^
                        } while ((Get-Date) -lt $deadline); ^
                        Write-Host "Health check timed out."; ^
                        exit 1"
                    '''

                    echo 'Verifying deployed EVAT API...'

                    bat '''
                        powershell -NoProfile -Command ^
                        "$response=Invoke-WebRequest -Uri 'http://localhost:8082/api/docs/' -UseBasicParsing; ^
                        Write-Host ('HTTP Status: ' + $response.StatusCode); ^
                        if ($response.StatusCode -ne 200) { exit 1 }"
                    '''

                    echo 'EVAT deployment completed successfully.'
                }
            }

            post {
                failure {
                    echo 'Deployment failed. Attempting rollback...'

                    bat 'docker rm -f evat-backend-staging 2>NUL || exit /b 0'

                    bat '''
                        powershell -NoProfile -Command ^
                        "$existing=docker ps -aq --filter name=evat-backend; ^
                        if ($existing) { ^
                            docker rm -f evat-backend 2>$null ^
                        }; ^
                        docker run -d --name evat-backend -p 8082:8081 --env-file server/node-api/.env -e MONGODB_URI=mongodb://host.docker.internal:27017/evat evat-backend:1.1"
                    '''

                    echo 'Rollback attempted using known-good evat-backend:1.1.'
                }

                success {
                    echo 'Deployment and health checks passed.'
                }
            }
        }
    }
}