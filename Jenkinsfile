pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Code successfully pulled from GitHub'
            }
        }

        stage('Test App Files') {
            steps {
                sh 'ls -la'
                sh 'cat app.py'
                sh 'cat Dockerfile'
                echo 'Jenkins CI stage is working'
            }
        }
    }
}
