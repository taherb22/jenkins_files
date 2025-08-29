pipeline {
    agent {
        label 'secure-agent'
    }

    environment {
        API_TOKEN = credentials('my-api-token')
        DISABLE_INSECURE_FEATURES = true
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // --- THIS IS THE MODIFIED PART ---
                    // The old parser cannot handle "?." or "=~ /.../"
                    if (params.BRANCH_NAME == null || params.BRANCH_NAME.trim().isEmpty()) {
                        error 'Branch name is required and cannot be empty'
                    }
                    if (params.BRANCH_NAME.matches(".*[^a-zA-Z0-9-_].*")) {
                        error 'Branch name contains invalid characters'
                    }
                }
                sh 'rm -rf * && mkdir -p build'
                echo "Building branch: ${params.BRANCH_NAME}"
            }
        }

        stage('Build') {
            steps {
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
                container('maven') {
                    sh 'mvn test -B'
                }
                junit 'target/surefire-reports/*.xml'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([string(credentialsId: 'deploy-api-key', variable: 'DEPLOY_KEY')]) {
                    sh '''
                        curl -X POST https://api.example.com/deploy \
                        -H "Authorization: Bearer $DEPLOY_KEY" \
                        -d "branch=${BRANCH_NAME}" \
                        --fail --silent
                    '''
                }
                sh 'curl https://example.com/health --fail --silent || exit 1'
            }
        }
    }

    post {
        always {
            cleanWs()
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
            }
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Build ${env.BUILD_NUMBER} Failed",
                 body: "Check Jenkins for details: ${env.BUILD_URL}"
        }
    }
}