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
                script {
                    // Capture the newman output
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
            // Archive the test results
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "newman",
                reportFiles: 'report.html',
                reportName: 'Postman Test Report'
            ])

            // Send Discord notification with test results
            script {
                // Parse newman output for statistics
                def stats = [
                    total: (env.NEWMAN_OUTPUT =~ /iterations\s+\[\d+\/(\d+)\]/)[0][1] as Integer,
                    failed: (env.NEWMAN_OUTPUT =~ /iterations\s+\[(\d+)\/\d+\]/)[0][1] as Integer,
                    failedTests: (env.NEWMAN_OUTPUT =~ /assertions\s+\[(\d+)\/\d+\]/)[0][1] as Integer,
                    totalTests: (env.NEWMAN_OUTPUT =~ /assertions\s+\[\d+\/(\d+)\]/)[0][1] as Integer
                ]
                
                // Calculate pass percentage
                def passPercentage = ((stats.totalTests - stats.failedTests) / stats.totalTests * 100).round(2)
                
                // Determine color based on results (green: all passed, yellow: some failed, red: all failed)
                def color
                if (stats.failedTests == 0) {
                    color = 65280  // Green
                } else if (stats.failedTests == stats.totalTests) {
                    color = 16711680  // Red
                } else {
                    color = 16776960  // Yellow
                }

                // Create Discord message embed
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
            }
        }
    }
}