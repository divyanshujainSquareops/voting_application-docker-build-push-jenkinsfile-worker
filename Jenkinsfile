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
        Git_clone_repo_url= "https://github.com/divyanshujainSquareops/voting_application-docker-build-push-jenkinsfile-worker.git"
        Git_helm_repo_url= "https://github.com/divyanshujainSquareops/voting_application-helm-argocd.git"
        branch_name= "main"
        credentialsId= "github"
        user_email= "divyanshu.jain@squareops.com"
        user_name= "Divyanshu Jain"
        
    }

    stages {
        stage('Clone in Kaniko Container') {
            steps {
                script {
                    container('kaniko') {
                        echo "${branch_name},${credentialsId},${Git_clone_repo_url}"
                        git branch: "${branch_name}", credentialsId: "${credentialsId}", url: "${Git_clone_repo_url}"
                        echo "Repository cloned inside Kaniko container"
                    }
                }
            }
        }
         stage('Build and Push vote Images') {
            steps {
                script {
                    container('kaniko') {
                        // Build and push Docker image using Kaniko
                        sh "/kaniko/executor --dockerfile `pwd`/vote/Dockerfile --context=`pwd` --destination=${DOCKER_HUB_REPO}/votingapp-vote:${BUILD_NUMBER}"
                        echo "Image vote build completed"
                    }
                }
            }
        }
        stage('Helm Values Update and Push') {
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
                    
                   
                    git branch: "${branch_name}", credentialsId: "${credentialsId}", url: "${Git_helm_repo_url}"
                     withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER_NAME', passwordVariable: 'PASSWORD')]) {
                        sh '''
                            cd ./vote/
                            yq e -i '.image.tag = "'$BUILD_NUMBER'"' values.yaml
                            cd ..
                            git config --global --add safe.directory /home/jenkins/agent/workspace/${JOB_NAME}
                            git config --global user.email ${user_email}
                            git config --global user.name ${user_name}
                            git add .
                            git commit -m "commit=\$JOB_NAME-\$BUILD_NUMBER-\$BUILD_DATE"
                            git push https://${GIT_USER_NAME}:${PASSWORD}@github.com/divyanshujainSquareops/voting_application-helm-argocd.git main
                        '''                      
                    }
                }
            }
          }
        }
    }
}
