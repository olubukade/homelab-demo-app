pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod

spec:
  serviceAccountName: jenkins
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
    image: alpine/k8s:1.30.0
    command:
    - cat
    tty: true

  - name: trivy
    image: aquasec/trivy:latest
    command:
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
                sh 'cat k8s/deployment.yaml'
            }
        }

        stage('SonarQube Code Scan') {
            steps {
                container('jnlp') {
                    script {
                        def scannerHome = tool 'SonarScanner'
                        withSonarQubeEnv('SonarQube') {
                            sh """
                                ${scannerHome}/bin/sonar-scanner
                            """
                        }
                    }
                }
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

        stage('Trivy Image Scan') {
            steps {
                container('trivy') {
                    sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --ignore-unfixed \
                      $IMAGE_NAME:latest
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