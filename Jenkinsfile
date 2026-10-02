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
    }
}