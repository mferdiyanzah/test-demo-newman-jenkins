pipeline {
    agent any

    tools {
        nodejs 'nodejs'
    }
    
    environment {
        NEWMAN_COLLECTION = 'postman/collection.json'
        NEWMAN_ENVIRONMENT = 'postman/env.json'
        NPM_CACHE = "${WORKSPACE}/.npm"
        DISCORD_WEBHOOK_URL = 'https://discord.com/api/webhooks/1298922886257578056/wCR-TDuvBpUz9zCZyQ8nXqXQiorOnLeEUcQ_SABnjp0ZW7eCEEhg0AADfrlW0b0J1d3D'
    }
    
    stages {
        stage('Create Report Directory') {
            steps {
                sh 'mkdir -p newman'
            }
        }

        stage('Setup Node') {
            steps {
                sh 'node --version'
                sh 'npm --version'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm config set cache $NPM_CACHE --global
                    npm install -g newman
                    npm install -g newman-reporter-htmlextra
                    newman --version
                '''
            }
        }
        
        stage('Run Postman Collection') {
            steps {
                sh '''
                    newman run ${NEWMAN_COLLECTION} \
                    --environment ${NEWMAN_ENVIRONMENT} \
                    --reporters cli,htmlextra \
                    --reporter-htmlextra-export newman/report.html \
                    --reporter-htmlextra-darkTheme \
                    --suppress-exit-code
                '''
            }
        }
    }
    
    post {
        always {
            // Archive the test results
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "newman",
                reportFiles: 'report.html',
                reportName: 'Postman Test Report'
            ])

            // Send Discord notification
            script {
                def message = "Build #${currentBuild.number} completed: ${currentBuild.currentResult}. Check the report here: ${env.BUILD_URL}"
                sh """
                    curl -H "Content-Type: application/json" -d '{"content": "${message}"}' ${DISCORD_WEBHOOK_URL}
                """
            }
        }
    }
}
