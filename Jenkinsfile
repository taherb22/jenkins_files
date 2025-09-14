pipeline {
    agent { label 'trusted-builder' }

    stages {
        stage('Build and Test') {
            steps {
                echo 'Running build and test procedures...'
                sh './run_tests.sh'
            }
        }
    }
}