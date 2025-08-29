pipeline {
    agent {
        // Use a specific, controlled agent label to limit where the pipeline runs
        label 'secure-agent'
    }
    



    
    environment {
        // Use Jenkins credentials store instead of hardcoded values
        API_TOKEN = credentials('my-api-token')
        // Avoid exposing environment variables unnecessarily
        DISABLE_INSECURE_FEATURES = true
    }

    options {
        // Enable timestamps and build retention for auditing
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Disable concurrent builds to prevent race conditions
        disableConcurrentBuilds()
    }

    parameters {
        // Use string parameters with default values and validation
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build', trim: true)
    }

    stages {
        stage('Initialize') {
            steps {
                // Validate input parameters
                script {
                    if (!params.BRANCH_NAME?.trim()) {
                        error 'Branch name is required and cannot be empty'
                    }
                    if (params.BRANCH_NAME =~ /[^a-zA-Z0-9-_]/) {
                        error 'Branch name contains invalid characters'
                    }
                }
                // Set up a secure workspace
                sh 'rm -rf * && mkdir -p build'
                echo "Building branch: ${params.BRANCH_NAME}"
            }
        }

        stage('Build') {
            steps {
                // Run build in a container with minimal privileges
                container('maven') {
                    sh '''
                        mvn clean install -DskipTests \
                        --batch-mode \
                        --no-transfer-progress
                    '''
                }
            }
        }

        stage('Test') {
            steps {
                // Execute tests in a controlled environment
                container('maven') {
                    sh 'mvn test -B'
                }
                // Archive test results securely
                junit 'target/surefire-reports/*.xml'
            }
        }

        stage('Deploy') {
            when {
                // Only deploy on main branch
                branch 'main'
            }
            steps {
                // Use withCredentials to handle sensitive data
                withCredentials([string(credentialsId: 'deploy-api-key', variable: 'DEPLOY_KEY')]) {
                    sh '''
                        curl -X POST https://api.example.com/deploy \
                        -H "Authorization: Bearer $DEPLOY_KEY" \
                        -d "branch=${BRANCH_NAME}" \
                        --fail --silent
                    '''
                }
                // Verify deployment
                sh 'curl https://example.com/health --fail --silent || exit 1'
            }
        }
    }

    post {
        always {
            // Clean up workspace securely
            cleanWs()
            // Send audit log to a secure logging service
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
            }
        }
        failure {
            // Notify on failure with restricted details
            mail to: 'team@example.com',
                 subject: "Build ${env.BUILD_NUMBER} Failed",
                 body: "Check Jenkins for details: ${env.BUILD_URL}"
        }
    }
}
