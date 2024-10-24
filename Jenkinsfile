pipeline {
    agent any

    tools {
        nodejs 'nodejs'
    }
    
    environment {
        NEWMAN_COLLECTION = 'postman/collection.json'
        NEWMAN_ENVIRONMENT = 'postman/env.json'
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
                // Cache npm packages
                cache(maxCacheSize: 500, path: '~/.npm', key: 'npm-cache') {
                    sh 'npm install -g newman || true'
                }
                // Cache global npm modules
                cache(maxCacheSize: 500, path: '/usr/local/lib/node_modules', key: 'npm-global') {
                    sh 'npm install -g newman-reporter-htmlextra || true'
                }
                // Verify installation
                sh 'newman --version'
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
        }
    }
}
