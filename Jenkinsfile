pipeline {
    agent any
    
    environment {
        NEWMAN_COLLECTION = '/postman/collection.json'
        NEWMAN_ENVIRONMENT = '/postman/env.json'
        REPORT_DIRECTORY = 'newman-reports'
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                // Install Newman (Postman CLI) if not already installed
                sh '''
                    npm install -g newman
                    npm install -g newman-reporter-htmlextra
                '''
            }
        }
        
        stage('Run Postman Collection') {
            steps {
                sh 'mkdir -p ${REPORT_DIRECTORY}'
                
                sh '''
                    newman run ${NEWMAN_COLLECTION} \
                    --environment ${NEWMAN_ENVIRONMENT} \
                    --reporters cli,htmlextra \
                    --reporter-htmlextra-export ${REPORT_DIRECTORY}/report.html \
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
                reportDir: "${REPORT_DIRECTORY}",
                reportFiles: 'report.html',
                reportName: 'Postman Test Report'
            ])
        }
    }
}