pipeline {
    agent any
    tools {
        maven 'maven' // Ensure the Maven installation name matches the one configured in Jenkins
    }
    environment {
        IMAGE_NAME = "springbootapp"
        IMAGE_TAG = "${BUILD_NUMBER}" // Use build number as version
        ACR_NAME = "jenkinsazure"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        FULL_IMAGE_NAME = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        TENANT_ID = "ec78375d-0db0-42cf-82a6-2e6403e95936"
        RESOURCE_GROUP = "Jenkins"
        AKS_CLUSTER = "springboot"
        K8S_NAMESPACE = "default"
        K8S_DEPLOYMENT = "springboot-app"
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'prod', url: 'https://github.com/bkrrajmali/newspring-pet-clininc.git'
            }
        }

        stage('Maven Compile') {
            steps {
                echo "This is Maven Compile Stage"
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                echo "This is Maven Test Stage"
                sh 'mvn test'
            }
        }

        stage('File System Scan By Trivy') {
            steps {
                echo "Trivy Scan Started"
                sh 'trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .'
            }
        }

        stage('Sonar Analysis') {
            environment {
                SCANNER_HOME = tool 'Sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.organization=bkrrajmali \
                    -Dsonar.projectName=SpringBootPet \
                    -Dsonar.projectKey=bkrrajmali_springbootpet \
                    -Dsonar.java.binaries=. \
                    -Dsonar.exclusions=**/trivy-fs-output.txt
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar'
                }
            }
        }

        stage('Maven Package') {
            steps {
                echo 'Maven package Started'
                sh 'mvn package'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build image from Dockerfile in the root directory
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Azure Login to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        az acr login --name $ACR_NAME
                        '''
                    }
                }
            }
        }

        stage('Tag and Push Image to ACR') {
            steps {
                script {
                    sh """
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                    docker push ${FULL_IMAGE_NAME}
                    """
                }
            }
        }

        stage('Azure Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    sh '''
                    az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                    az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
                    '''
                }
            }
        }

        stage('Deploy to AKS (Create or Rolling Update)') {
            steps {
                script {
                    echo "🚀 Deploying to AKS: Check if deployment exists"
                    sh """
                    kubectl apply -f k8s/springboot-pvc.yaml -n $K8S_NAMESPACE

                    if kubectl get deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} > /dev/null 2>&1; then
                      echo "🔄 Deployment exists. Performing rolling update..."
                      kubectl set image deployment/${K8S_DEPLOYMENT} ${IMAGE_NAME}=${FULL_IMAGE_NAME} -n ${K8S_NAMESPACE}
                      kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}
                    else
                      echo "🆕 Deployment does not exist. Creating deployment..."
                      kubectl apply -f k8s/deployment-with-tag.yaml -n ${K8S_NAMESPACE}
                      kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}
                    fi
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ Deployment successful: ${FULL_IMAGE_NAME}"
            }
        }
        failure {
            echo "❌ Deployment failed. Please check the logs."
        }
    }
}

