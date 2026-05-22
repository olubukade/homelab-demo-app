pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod

spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest

    command:
    - sleep

    args:
    - 9999999

    tty: true

    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker

  volumes:
  - name: docker-config
    secret:
      secretName: regcred
"""
        }
    }

    environment {
        IMAGE_NAME = "docker.io/olubukade95/homelab-demo-app"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                sh 'ls -la'
                sh 'cat Dockerfile'
            }
        }

        stage('Build and Push Image') {
            steps {

                container('kaniko') {

                    sh '''
                    /kaniko/executor \
                      --context=$(pwd) \
                      --dockerfile=Dockerfile \
                      --destination=$IMAGE_NAME:latest
                    '''
                }
            }
        }
    }
}