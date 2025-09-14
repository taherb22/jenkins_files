pipeline {
    agent any // Vulnerability 1: Non-restrictive agent

    environment {
        AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE" // Vulnerability 2: Hardcoded Secret
    }

    stages {
        stage('Deploy') {
            steps {
                script {
                    // Vulnerability 3: Potential for Command Injection
                    sh "ansible-playbook -i inventory.ini deploy.yml --extra-vars 'version=${params.VERSION}'"
                }
            }
        }
    }

}    