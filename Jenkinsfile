pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod

spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/sh
    args:
    - -c
    - cat
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - /bin/sh
    args:
    - -c
    - cat
    tty: true

  volumes:
  - name: docker-config
    secret:
      secretName: regcred
      items:
      - key: .dockerconfigjson
        path: config.json

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
        stage('Deploy to Kubernetes') {
    steps {
        container('kubectl') {
            sh '''
            kubectl apply -f k8s/deployment.yaml
            kubectl rollout status deployment homelab-demo
            '''
        }
    }
}
            }
        }  
    }
}

