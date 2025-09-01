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
                if (params.BRANCH_NAME == null || params.BRANCH_NAME.trim().isEmpty()) {
                        error 'Branch name is required and cannot be empty'
                }
            }
        }
    }




}

    
