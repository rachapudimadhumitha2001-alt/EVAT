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
                    def scannerHome = tool 'SonarQube-Scanner'
                    bat "\"${scannerHome}\\bin\\sonar-scanner.bat\""
                }
            }
        }
    }
}