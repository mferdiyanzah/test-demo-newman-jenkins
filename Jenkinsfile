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
                script {
                    // Check if newman is installed
                    def newmanInstalled = sh(
                        script: 'newman --version >/dev/null 2>&1',
                        returnStatus: true
                    ) == 0
                    
                    // Check if newman-reporter-htmlextra is installed
                    def htmlextraInstalled = sh(
                        script: 'npm list -g newman-reporter-htmlextra >/dev/null 2>&1',
                        returnStatus: true
                    ) == 0
                    
                    sh 'npm config set cache $NPM_CACHE --global'
                    
                    if (!newmanInstalled) {
                        sh 'npm install -g newman'
                    }
                    
                    if (!htmlextraInstalled) {
                        sh 'npm install -g newman-reporter-htmlextra'
                    }
                    
                    sh 'newman --version'
                }
            }
        }
        
        stage('Run Postman Collection') {
            steps {
                script {
                    // Run Newman with both HTML and JSON reporters
                    sh '''
                        newman run ${NEWMAN_COLLECTION} \
                        --environment ${NEWMAN_ENVIRONMENT} \
                        --reporters cli,htmlextra,json \
                        --reporter-htmlextra-export newman/report.html \
                        --reporter-json-export newman/report.json \
                        --reporter-htmlextra-darkTheme \
                        --reporter-cli-no-console \
                        --suppress-exit-code || true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "newman",
                reportFiles: 'report.html',
                reportName: 'Postman Test Report'
            ])

            script {
                try {
                    // Read and parse the JSON report
                    def reportJson = readJSON file: 'newman/report.json'
                    
                    // Extract test results from JSON
                    def run = reportJson.run
                    def stats = run.stats
                    def assertions = stats.assertions
                    
                    def totalTests = assertions.total
                    def failedTests = assertions.failed
                    def passedTests = assertions.total - assertions.failed
                    def skippedTests = assertions.skipped ?: 0
                    
                    // Calculate pass percentage
                    def passPercentage = totalTests > 0 ? 
                        (passedTests * 100.0 / totalTests).setScale(2, java.math.RoundingMode.HALF_UP) : 
                        0
                    
                    // Determine color based on results
                    def color
                    if (failedTests == 0) {
                        color = 65280  // Green
                    } else if (passedTests == 0) {
                        color = 16711680  // Red
                    } else {
                        color = 16776960  // Yellow
                    }

                    def payload = """
                    {
                        "embeds": [{
                            "title": "Postman Test Results - Build #${currentBuild.number}",
                            "color": ${color},
                            "fields": [
                                {
                                    "name": "Status",
                                    "value": "${currentBuild.currentResult}",
                                    "inline": true
                                },
                                {
                                    "name": "Pass Rate",
                                    "value": "${passPercentage}%",
                                    "inline": true
                                },
                                {
                                    "name": "Test Results",
                                    "value": "✅ Passed: ${passedTests}\\n❌ Failed: ${failedTests}\\n⏩ Skipped: ${skippedTests}\\n📊 Total: ${totalTests}",
                                    "inline": false
                                }
                            ],
                            "footer": {
                                "text": "View detailed report: [Jenkins](${env.BUILD_URL})"
                            },
                            "timestamp": "${new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('UTC'))}"
                        }]
                    }
                    """

                    sh """
                        curl -H "Content-Type: application/json" -d '${payload}' ${DISCORD_WEBHOOK_URL}
                    """
                } catch (Exception e) {
                    echo "Error processing newman results: ${e.getMessage()}"
                    
                    def errorPayload = """
                    {
                        "embeds": [{
                            "title": "Postman Test Results - Build #${currentBuild.number}",
                            "color": 16711680,
                            "description": "Error occurred while processing test results: ${e.getMessage()}",
                            "fields": [
                                {
                                    "name": "Status",
                                    "value": "${currentBuild.currentResult}",
                                    "inline": true
                                }
                            ],
                            "footer": {
                                "text": "View detailed report: ${env.BUILD_URL}"
                            }
                        }]
                    }
                    """
                    
                    sh """
                        curl -H "Content-Type: application/json" -d '${errorPayload}' ${DISCORD_WEBHOOK_URL}
                    """
                }
            }
        }
    }
}