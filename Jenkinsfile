pipeline {
    agent { label 'docker-builder' }
    
    stages {
        stage('Build') {
            steps {
                sh 'docker build . -t my-app:latest'
            }
        }
        stage('Deploy to Production') {
            when { 
                expression { params.CONFIRM == true } // Flaw: No check on the branch name!
            }
            steps {
                echo 'Deploying to Production Environment!'
                sh 'kubectl apply -f production.yaml'
            }
        }
    }
    parameters {
        booleanParam(name: 'CONFIRM', defaultValue: false, description: 'Confirm deployment to production')
    }
}