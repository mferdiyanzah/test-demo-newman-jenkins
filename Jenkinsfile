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
                    env.NEWMAN_OUTPUT = sh(
                        script: '''
                            newman run ${NEWMAN_COLLECTION} \
                            --environment ${NEWMAN_ENVIRONMENT} \
                            --reporters cli,htmlextra \
                            --reporter-htmlextra-export newman/report.html \
                            --reporter-htmlextra-darkTheme \
                            --reporter-cli-no-console \
                            --suppress-exit-code 2>&1
                        ''',
                        returnStdout: true
                    ).trim()
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
                    def stats = [
                        total: 0,
                        failed: 0,
                        failedTests: 0,
                        totalTests: 0
                    ]
                    
                    // Safely extract statistics with null checks
                    def newmanOutput = env.NEWMAN_OUTPUT ?: ''
                    
                    // More robust regex patterns with defensive programming
                    def iterationMatch = newmanOutput =~ /iterations\s+\[(\d+)\/(\d+)\]/
                    def assertionMatch = newmanOutput =~ /assertions\s+\[(\d+)\/(\d+)\]/
                    
                    if (iterationMatch.find()) {
                        stats.failed = iterationMatch.group(1) as Integer
                        stats.total = iterationMatch.group(2) as Integer
                    }
                    
                    if (assertionMatch.find()) {
                        stats.failedTests = assertionMatch.group(1) as Integer
                        stats.totalTests = assertionMatch.group(2) as Integer
                    }
                    
                    // Safe calculation of pass percentage
                    def passPercentage = stats.totalTests > 0 ? 
                        ((stats.totalTests - stats.failedTests) / stats.totalTests * 100).round(2) : 
                        0
                    
                    // Determine color based on results
                    def color = 16776960 // Default to Yellow
                    if (stats.totalTests > 0) {
                        if (stats.failedTests == 0) {
                            color = 65280  // Green
                        } else if (stats.failedTests == stats.totalTests) {
                            color = 16711680  // Red
                        }
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
                                    "value": "✅ Passed: ${stats.totalTests - stats.failedTests}\\n❌ Failed: ${stats.failedTests}\\n📊 Total: ${stats.totalTests}",
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
                    // Send a simplified Discord notification in case of error
                    def errorPayload = """
                    {
                        "embeds": [{
                            "title": "Postman Test Results - Build #${currentBuild.number}",
                            "color": 16711680,
                            "description": "Error occurred while processing test results",
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