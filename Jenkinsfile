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
                    sh '''
                        newman run ${NEWMAN_COLLECTION} \
                        --environment ${NEWMAN_ENVIRONMENT} \
                        --reporters cli,htmlextra \
                        --reporter-htmlextra-export newman/report.html \
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
                    // Read the HTML report file
                    def htmlContent = readFile('newman/report.html')
                    
                    // Parse total passed and failed tests from the footer
                    def totalPassed = 0
                    def totalFailed = 0
                    def totalSkipped = 0
                    
                    // Look for the footer row with totals
                    def footerMatch = htmlContent =~ /<tr class="bg-light">[^<]*<td><strong>Total<\/strong><\/td>[^<]*<td class="text-center">(\d+)<\/td>[^<]*<td class="text-center">(\d+)<\/td>[^<]*<td class="text-center">(\d+)<\/td>/
                    
                    if (footerMatch.find()) {
                        totalPassed = footerMatch[0][1] as Integer
                        totalFailed = footerMatch[0][2] as Integer
                        totalSkipped = footerMatch[0][3] as Integer
                    }
                    
                    def totalTests = totalPassed + totalFailed + totalSkipped
                    def passPercentage = totalTests > 0 ? ((totalPassed / totalTests) * 100).round(2) : 0
                    
                    // Determine color based on results
                    def color
                    if (totalFailed == 0) {
                        color = 65280  // Green
                    } else if (totalPassed == 0) {
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
                                    "value": "✅ Passed: ${totalPassed}\\n❌ Failed: ${totalFailed}\\n⏩ Skipped: ${totalSkipped}\\n📊 Total: ${totalTests}",
                                    "inline": false
                                }
                            ],
                            "footer": {
                                "text": "View detailed report: ${env.BUILD_URL}"
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