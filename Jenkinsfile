pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Code checked out from GitHub'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t homelab-demo:latest .'
            }
        }

        stage('Import Image into K3s') {
            steps {
                sh 'docker save homelab-demo:latest | sudo k3s ctr images import -'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl rollout restart deployment homelab-demo'
                sh 'kubectl get pods'
            }
        }
    }
}
