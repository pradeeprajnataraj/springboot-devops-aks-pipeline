pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment {
        IMAGE_NAME = "springboot"
        IMAGE_TAG = "latest"
        ACR_NAME = "dockerregnodejs"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        FULL_IMAGE_NAME = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        TENANT_ID = "ec78375d-0db0-42cf-82a6-2e6403e95936"
        RESOURCE_GROUP = "demo-rg"
        CLUSTER_NAME = "demo-eks"
    }
    stages {
        stage('Checkout From Git') { 
            steps {
                git branch: 'main', url: 'https://github.com/pradeeprajnataraj/springboot-devops-aks-pipeline.git'
            }
        }

        stage('Maven Package') {
            steps {
                echo 'Packaging application...'
                sh 'mvn package'
            }
        }

        stage('Sonar Analysis') {
            environment {
                SCANNER_HOME = tool 'SonarScanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=enhanced-petclinic-prod \
                        -Dsonar.projectName=enhanced-petclinic-prod \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target \
                        -Dsonar.exclusions=**/trivy-report.txt
                    '''
                }
            }
        }

        stage('Docker Build') { 
            steps {
                script {
                    echo 'Creating Docker Image'
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Azure Login to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azurespn', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        az acr login --name $ACR_NAME
                        '''
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    echo 'Pushing Docker image to registry'
                    sh '''
                    docker tag "${IMAGE_NAME}:${IMAGE_TAG}" ${FULL_IMAGE_NAME}
                    docker push ${FULL_IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Jenkins Login to AKS Cluster') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azurespn', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        sh '''
                        echo "Logging into AKS cluster"
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to AKS'
                    sh '''
                    kubectl apply -f k8s/springboot-deployment.yaml
                    echo 'Deployment Done'
                    '''
                }
            }
        }
    }
}
