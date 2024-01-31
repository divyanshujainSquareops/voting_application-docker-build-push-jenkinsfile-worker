pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
    name: kaniko
spec:
    restartPolicy: Never
    volumes:
    - name: kaniko-secret
      secret:
            secretName: kaniko-secret
    containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command:
        - /busybox/cat
      tty: true
      volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
"""
        }
    }

    environment {
        DOCKER_HUB_REPO = 'divyanshujain11'
        BUILD_DATE = sh(script: 'date "+%Y-%m-%d"', returnStdout: true).trim()
    }

    stages {
        stage('Worker-repo Clone in Kaniko Container') {
            steps {
                script {
                    container('kaniko') {
                        git branch: 'main', credentialsId: 'github', url: 'https://github.com/divyanshujainSquareops/voting_application-docker-build-push-jenkins.git'
                        echo "Repository cloned inside Kaniko container"
                    }
                }
            }
        }
        stage('Build and Push worker Images') {
            steps {
                script {
                    container('kaniko') {
                        // Build and push Docker image using Kaniko
                        sh "/kaniko/executor --dockerfile `pwd`/worker/Dockerfile --context=`pwd` --destination=${DOCKER_HUB_REPO}/votingapp-worker:${BUILD_NUMBER}"
                        echo "worker Image build completed"
                    }
                }
            }
        }
        stage('Worker Helm Values Update and Push') {
          agent {
            kubernetes {
                yaml """
                apiVersion: v1
                kind: Pod
                metadata:
                    name: builder
                spec:
                  restartPolicy: Never
                  volumes:
                    - name: builder-storage
                      emptyDir: {}
                  containers:
                  - name: builder
                    image: squareops/jenkins-build-agent:v3
                    securityContext:
                      privileged: true
                    volumeMounts:
                      - name: builder-storage
                        mountPath: /var/lib/docker
                """
            }
          }
          steps {
            script {
                container('builder') {
                    // Clone Git repo with credentials
                    
                   
                    git branch: 'main', credentialsId: 'github', url: 'https://github.com/divyanshujainSquareops/voting_application-helm-argocd.git'
                     withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
                        sh '''
                            cd ./worker/
                            yq e -i '.image.tag = "'$BUILD_NUMBER'"' values.yaml
                            cd ..
                            git config --global --add safe.directory /home/jenkins/agent/workspace/${JOB_NAME}
                            git config --global user.email "divyanshu.jain@squareops.com"
                            git config --global user.name "Divyanshu jain"
                            git add .
                            git commit -m "commit=\$JOB_NAME-\$BUILD_NUMBER-\$BUILD_DATE"
                            git push https://${USER_NAME}:${PASSWORD}@github.com/divyanshujainSquareops/voting_application-helm-argocd.git main
                        '''                      
                    }
                }
            }
          }
        }
    }
}
