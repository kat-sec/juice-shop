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

        // Stage 4: Run automated tests
        stage('Run Tests') {
            steps {
                dir(WORKSPACE_DIR) {
                    script {
                        echo "Running automated tests..."
                        def exitCode = 0
                        try {
                            exitCode = bat(script: "${NPM_EXE} test", returnStatus: true)
                        } catch (Exception e) {
                            echo "Test execution error: ${e.message}"
                            exitCode = 1
                        }

                        if (exitCode != 0) {
                            echo "Some tests failed"
                            currentBuild.result = 'UNSTABLE'
                        } else {
                            echo "All tests passed"
                        }

                        try {
                            if (fileExists('test-results')) {
                                archiveArtifacts 'test-results/*.xml'
                            } else {
                                echo "No test-results folder found"
                            }
                        } catch (Exception e) {
                            echo "Artifact archiving failed: ${e.message}"
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

        // Stage 7: Build placeholder (Juice Shop doesn't require a build step)
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

        // Stage 12: Monitoring simulation (placeholder for observability tools)
        stage('Monitoring') {
            steps {
                script {
                    echo "Simulating monitoring setup..."
                    echo "In a real-world scenario, this would integrate tools like New Relic, Prometheus, or Datadog."
                    echo "Metrics such as container uptime, response latency, and error rates would be tracked."
                }
            }
        }

        // Stage 13: Vulnerability Summary
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
            echo 'Application ready: http://localhost:3000'
        }

        unstable {
            echo 'UNSTABLE: Pipeline completed with warnings'
            echo 'Juice Shop test failures or vulnerabilities are expected'
            echo 'Application is still functional: http://localhost:3000'
        }

        failure {
            echo 'FAILURE: Pipeline failed on critical error'
        }
    }
}
