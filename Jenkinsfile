pipeline {
    agent any
    
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'd32d1722-e925-4443-8b87-1f5a2617733b', url: 'https://github.com/hamza-shabbir94/boardgame-main.git'
            }
        }
        stage('Gitleaks') {
            steps {
                sh 'gitleaks detect --source . -r gitleaks-reports.txt --redact'
            }
        }
        
        
        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('trivy fs scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        stage('unit test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') { // sonar is server which we configure in the jenkins system config
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Boardgame \
                    -Dsonar.projectKey=Boardgame -Dsonar.java.binaries=target '''
                }
            }
        }
        stage('Quality Gate check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Deploy to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'gmavensetting', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }
        stage('Docker Build & tag') {
            steps {
                script {
                    // This step should not normally be used in your script. Consult the inline help for details.
                    withDockerRegistry(credentialsId: 'docker-cred') {
                      sh 'set +x && docker build -t hamza94/boardgame:latest .'
                    }
                }
            }
        }
        stage('trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html hamza94/boardgame:latest'
            }
        }
        stage('Docker Push Image') {
            steps {
                script {
                    // This step should not normally be used in your script. Consult the inline help for details.
                    withDockerRegistry(credentialsId: 'docker-cred') {
                      sh 'docker push hamza94/boardgame:latest'
                    }
                }
            }
        }
        stage('deploy to kubernetes') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-token', namespace: 'default', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.43.20:6443') {
                         sh 'kubectl apply -f deployment-service.yaml'
                         sh 'kubectl get all'
                    }
                }
            }
        }
    }
}
