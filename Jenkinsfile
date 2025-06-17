pipeline {
    agent any
    tools {
        maven 'maven'
    }
    stages {
        stage('Checkout From Git') { 
            steps {
                git branch: 'prod', url: 'https://github.com/bkrrajmali/enahanced-petclinc-springboot.git'
            }
        }
        stage('Maven Compile') { 
            steps {
                echo 'This Maven Compile Stage'
                sh 'mvn compile'
            }
        }
        stage('Maven Test') { 
            steps {
                echo 'This Maven Test Stage'
                sh 'mvn test'
            }
        }
        stage('File System Scan By Trivy') { 
            steps {
                echo 'Trivy Scanning Started'
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
                          -Dsonar.exclusions=**/trivy-report.txt
                    '''
                }
            }
        }
        stage('Sonar Quality Gate') { 
            steps {
                echo 'Sonar Quality Gate Stage Started'
                steps {
                    timeout(time: 1, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true, credentialsId: 'sonar'
                    }
                }
            }
        }
    }
}




