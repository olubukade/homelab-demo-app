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
    image: dtzar/helm-kubectl:3.14.4
    command:
    - cat
    tty: true

  - name: trivy
    image: aquasec/trivy:latest
    command:
    - cat
    tty: true

  - name: python
    image: python:3.12-slim
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
        APP_URL = "http://10.0.0.124:32361"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                sh '''
                ls -la
                cat app.py
                cat Dockerfile
                cat requirements.txt
                cat requirements-dev.txt
                cat sonar-project.properties
                cat helm/homelab-demo-app/values.yaml
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                container('python') {
                    sh '''
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install -r requirements-dev.txt
                    '''
                }
            }
        }

        stage('Unit Tests') {
            steps {
                container('python') {
                    sh '''
                    PYTHONPATH=. pytest -v
                    '''
                }
            }
        }

        stage('Code Coverage') {
            steps {
                container('python') {
                    sh '''
                    PYTHONPATH=. pytest --cov=. --cov-report=xml
                    ls -la coverage.xml
                    '''
                }
            }
        }

        stage('Trivy SCA Filesystem Scan') {
            steps {
                container('trivy') {
                    sh '''
                    trivy fs \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --scanners vuln,secret \
                      .
                    '''
                }
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

        stage('Quality Gate') {
            steps {
                container('jnlp') {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
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
                      --destination=$IMAGE_NAME:$BUILD_NUMBER
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
                      --exit-code 0 \
                      --ignore-unfixed \
                      $IMAGE_NAME:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Update Helm Image Tag') {
            steps {
                container('jnlp') {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                     )]) {
                        sh '''
                        sed -i "s/tag: .*/tag: $BUILD_NUMBER/" helm/homelab-demo-app/values.yaml

                        git config user.email "jenkins@homelab.local"
                        git config user.name "Jenkins CI"

                        git add helm/homelab-demo-app/values.yaml
                        git commit -m "Update image tag to build $BUILD_NUMBER" || echo "No changes to commit"

                        git push https://$GIT_USERNAME:$GIT_TOKEN@github.com/olubukade/homelab-demo-app.git HEAD:main
                        '''
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                container('kubectl') {
                    sh '''
                    echo "Waiting for ArgoCD/Kubernetes deployment to complete..."
                    sleep 45

                    echo "Testing application health endpoint..."
                    curl -f $APP_URL/health
                    '''
                }
            }
        }
    }

    post {

        success {
            slackSend(
                channel: '#devops-alerts',
                color: 'good',
                tokenCredentialId: 'slack-token',
                botUser: true,
                message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} completed successfully. ArgoCD will deploy from Git. ${env.BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: '#devops-alerts',
                color: 'danger',
                tokenCredentialId: 'slack-token',
                botUser: true,
                message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. ${env.BUILD_URL}"
            )
        }
    }
}