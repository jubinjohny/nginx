pipeline {
    agent any
    environment {
        ACCOUNTID = """${sh(
                returnStdout: true,
                script: "curl -s  http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.accountId'"
                ).trim()}"""
        REGION = """${sh(
                returnStdout: true,
                script: "curl -s  http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region'"
                ).trim()}"""
        REPO_NAME = JOB_BASE_NAME.toLowerCase()
    }
    stages {
        stage('Sanity') {
            steps {
                echo 'Listing all files and checking sanity of files and build environment'
                sh "ls -l"
                sh "printenv"
            }
        }
        stage('Test') {
            steps {
                echo 'Run tests if any'
            }
        }
        stage('Build') {
            steps {
                echo 'Building docker image'
                sh "sudo docker build -t ${REPO_NAME}:${GIT_COMMIT} ."
            }
        }
        stage('Create REPO')  {
            steps {
                catchError(buildResult: 'SUCCESS')  {
                    sh "aws ecr describe-repositories --repository-name ${REPO_NAME}"
                }
                catchError(buildResult: 'SUCCESS')  {
                    sh "aws ecr create-repository --repository-name ${REPO_NAME}"
                }
            }
        }
        stage('Push') {
            steps {
                echo 'Pushing docker image to ECR. if this step fails, make sure that the repository with the same name as the repo name exists in ECR'
                sh "sudo docker tag ${REPO_NAME}:${GIT_COMMIT} ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}:${GIT_COMMIT}"
                sh "sudo docker tag ${REPO_NAME}:${GIT_COMMIT} ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}:${BUILD_NUMBER}"
                sh "aws ecr get-login-password --region ${REGION} | sudo docker login --username AWS --password-stdin ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com"
                sh "sudo docker push ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}:${GIT_COMMIT}"
                sh "sudo docker push ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}:${BUILD_NUMBER}"
            }
        }
        stage('Setup Helm Charts') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ubuntu', keyFileVariable: 'SSH_KEY')]) {
                catchError(buildResult: 'SUCCESS')  {
                    sh '''
                        rm -rf drinkprime-helm-charts && GIT_SSH_COMMAND="ssh -i $SSH_KEY" git clone git@bitbucket.org:drinkprimetechnology/drinkprime-helm-charts.git && ls -al
                        '''
                }
                }
            }
        }
        stage('Deploy') {
            steps {
                sh "aws eks update-kubeconfig --name kubernetes-cluster"
                sh "helm version"
                sh "kubectl get nodes"
                echo 'Deploying application to kubernetes using Helm'
                sh "helm upgrade ${REPO_NAME} drinkprime-helm-charts/charts/services -f values.yaml --set image.tag=${BUILD_NUMBER} --create-namespace -n ${REPO_NAME} --install "
            }
        }
    }
}