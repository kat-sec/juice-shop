pipeline {
    agent any

    environment {
        // Node.js setup
        NODE_HOME = 'C:\\Program Files\\Nodejs'
        PATH = "${NODE_HOME};${env.PATH}"

        // Executable paths
        GIT_EXE = '"C:\\Program Files\\Git\\bin\\git.exe"'
        NPM_EXE = 'npm'

        // Project-specific variables
        WORKSPACE_DIR = 'juice-shop'
        DOCKER_IMAGE = 'juice-shop'
    }

    stages {
        // Stage 1: Clean up any previous workspace
        stage('Clean Workspace') {
            steps {
                script {
                    if (fileExists(WORKSPACE_DIR)) {
                        echo "Deleting old workspace folder..."
                        bat "rmdir /s /q ${WORKSPACE_DIR}"
                    }
                }
            }
        }

        // Stage 2: Clone Juice Shop repository
        stage('Checkout') {
            steps {
                script {
                    try {
                        bat "${GIT_EXE} clone https://github.com/juice-shop/juice-shop.git ${WORKSPACE_DIR}"
                    } catch (Exception e) {
                        echo "Checkout failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        // Stage 3: Install Node.js dependencies
        stage('Install Dependencies') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        try {
                            echo "Installing Node.js dependencies..."
                            bat "${NPM_EXE} install"
                        } catch (Exception e) {
                            echo "Dependency installation failed: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
    // Create Test Results Folder
stage('Prepare Test Results Folder') {
    steps {
        dir(WORKSPACE_DIR) {
            script {
                if (!fileExists('test-results')) {
                    echo "Creating test-results folder..."
                    bat "mkdir test-results"
                } else {
                    echo "test-results folder already exists"
                }
            }
        }
    }
}

// Stage 4: Run automated tests
stage('Run Tests') {
    steps {
        dir(WORKSPACE_DIR) {
            script {
                echo "Running automated tests with JUnit reporter..."

                // Run tests and generate XML for Jenkins
                def exitCode = bat(
                    script: "${NPM_EXE} test -- --reporter mocha-junit-reporter --reporter-options mochaFile=test-results/results.xml",
                    returnStatus: true
                )

                if (exitCode != 0) {
                    echo "Some tests failed"
                    currentBuild.result = 'UNSTABLE'
                } else {
                    echo "ll tests passed"
                }

                // Archive XML artifacts
                if (fileExists('test-results/results.xml')) {
                    archiveArtifacts 'test-results/*.xml'
                } else {
                    echo "No XML results found, skipping archive."
                }
            }
        }
    }
}


        // Stage 5: Linting and security audit
        stage('Code Quality Analysis') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        echo "Running static code analysis..."

                        try {
                            def lintExit = bat(script: "${NPM_EXE} run lint", returnStatus: true)
                            if (lintExit != 0) {
                                echo "Linting issues detected"
                                currentBuild.result = 'UNSTABLE'
                            }
                        } catch (Exception e) {
                            echo "Linting error: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }

                        try {
                            def auditExit = bat(script: "${NPM_EXE} audit", returnStatus: true)
                            if (auditExit != 0) {
                                echo "Security vulnerabilities found"
                                currentBuild.result = 'UNSTABLE'
                            }
                        } catch (Exception e) {
                            echo "Audit error: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        // Stage 6: Deep security scanning
        stage('Security Scan') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        echo "Running security scans..."
                        try {
                            bat "${NPM_EXE} audit --audit-level=moderate"
                        } catch (Exception e) {
                            echo "Audit scan failed: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }

                        try {
                            bat "npx snyk test || echo Snyk scan completed"
                        } catch (Exception e) {
                            echo "Snyk scan error: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        // Stage 7: Build placeholder since Juice Shop doesn't require a build step
        stage('Build') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        echo "Running placeholder build step..."
                        try {
                            bat "${NPM_EXE} run build"
                        } catch (Exception e) {
                            echo "Build error: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        // Stage 8: Build Docker image
        stage('Docker Build') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        echo "Building Docker image..."
                        try {
                            bat "docker build -t ${DOCKER_IMAGE}:latest ."
                        } catch (Exception e) {
                            echo "Docker build failed: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        // Stage 9: Deploy container locally for testing
        stage('Deploy to Test Environment') {
            steps {
                script {
                    try {
                        bat """
                            docker stop ${DOCKER_IMAGE}-test || echo No container to stop
                            docker rm ${DOCKER_IMAGE}-test || echo No container to remove
                            docker run -d -p 3000:3000 --name ${DOCKER_IMAGE}-test ${DOCKER_IMAGE}:latest
                            ping -n 30 127.0.0.1 >nul
                        """
                    } catch (Exception e) {
                        echo "Deployment failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        // Stage 10: Verify application is running
        stage('Test Deployment') {
            steps {
                script {
                    echo "Testing application health..."
                    try {
                        def result = bat(script: "curl -s http://localhost:3000", returnStdout: true).trim()
                        if (result.contains('OWASP Juice Shop') || result.contains('juice shop')) {
                            echo "Application deployed successfully"
                        } else {
                            echo "Application may not be fully started. Response: ${result.take(100)}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    } catch (Exception e) {
                        echo "Deployment test failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        // Stage 11: Tag Docker image for release
        stage('Release Preparation') {
            steps {
                script {
                    try {
                        bat """
                            docker tag ${DOCKER_IMAGE}:latest ${DOCKER_IMAGE}:prod-${env.BUILD_NUMBER}
                            echo Release artifact: ${DOCKER_IMAGE}:prod-${env.BUILD_NUMBER}
                        """
                    } catch (Exception e) {
                        echo "Release tagging failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

// STAGE 12: Simulate Production Deployment
stage('Production Deployment Simulation') {
    steps {
        script {
            echo "Simulating production deployment using AWS CodeDeploy/Octopus Deploy..."
            echo "In a real environment, this would:"
            echo "1. Deploy to production AWS/Azure environment"
            echo "2. Use blue-green deployment strategy"
            echo "3. Perform health checks and validation"
            echo "4. Update production load balancers"
            
            // Simulate the deployment command that would be used
            bat """
                echo "Simulating: aws deploy create-deployment --application-name JuiceShopApp --deployment-group-name Production --revision-type S3 --s3-location bucket=my-app-bucket,key=juice-shop-${env.BUILD_NUMBER}.zip,bundleType=zip"
                echo "Simulating: octopus deploy release create --project JuiceShop --version ${env.BUILD_NUMBER} --deployTo Production"
            """
        }
    }
}
// Stage 13: Monitoring 
stage('Monitoring') {
    steps {
        script {
            echo "Collecting Docker container metrics for Juice Shop..."

            try {
                // Capture container stats once (no streaming)
                def stats = bat(
                    script: "docker stats ${DOCKER_IMAGE}-test --no-stream --format \"table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\"",
                    returnStdout: true
                ).trim()

                echo "Container Metrics:\n${stats}"

            } catch (Exception e) {
                echo "Monitoring failed: ${e.message}"
                currentBuild.result = 'UNSTABLE'
            }
        }
    }
}
// Production Monitoring Simulation
stage('Production Monitoring Simulation') {
    steps {
        script {
            echo "Simulating production monitoring with Datadog/New Relic..."
            echo "Monitoring would track:"
            echo "- Application performance (response times, error rates)"
            echo "- Infrastructure health (CPU, memory, disk usage)"
            echo "- Business metrics (user activity, transactions)"
            echo "- Security events (failed logins, suspicious activity)"
            
            // Simulate connecting to monitoring tools
            bat """
                echo "Simulating: datadog-monitor config --app JuiceShop --metrics cpu,memory,response_time"
                echo "Simulating: newrelic deployment create --appId JUICESHOP --revision ${env.BUILD_NUMBER}"
                echo "Simulating: alerts would be sent to Slack/Email/PagerDuty"
            """
            
            // Monitor something real using test container
            bat "docker stats ${DOCKER_IMAGE}-test --no-stream || echo 'Monitoring completed'"
        }
    }
}

        // Stage 14: Vulnerability Summary
        stage('Vulnerability Summary') {
            steps {
                script {
                    echo "Summarizing known vulnerabilities..."
                    echo "- Juice Shop is intentionally vulnerable for training purposes."
                    echo "- Critical issues found in vm2, tar, and ws packages."
                    echo "- Fixes are available but may introduce breaking changes."
                    echo "These vulnerabilities are acknowledged and acceptable in this context."
                }
            }
        }
    }

    post {
        always {
            script {
                try {
                    bat "docker stop ${DOCKER_IMAGE}-test || echo No container to stop"
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.message}"
                }
                try {
                    bat "docker rm ${DOCKER_IMAGE}-test || echo No container to remove"
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.message}"
                }

                archiveArtifacts allowEmptyArchive: true,
                    artifacts: '**/dist/*, **/coverage/*, **/test-results/*'
            }
        }

        success {
            echo 'SUCCESS: Pipeline completed'
        }

        unstable {
            echo 'UNSTABLE: Pipeline completed with warnings'
            echo 'Juice Shop test failures or vulnerabilities are expected'
        }

        failure {
            echo 'FAILURE: Pipeline failed on critical error'
        }
    }
    // For academic demonstration purposes, these stages show the complete automation workflow and integration points. In a commercial environment, the tagged Docker image would be deployed to AWS ECS, and Datadog would provide real-time monitoring with alerts routed to the operations team, 
    // the complete automation workflow and integration points. In a commercial environment, the tagged Docker image would be deployed to AWS ECS, and Datadog would provide real-time monitoring with alerts routed to the operations team.
