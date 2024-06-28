pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        RECIPIENTS = 'arifsadiq@gmail.com'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git credentialsId: 'github-cred', url: 'https://github.com/arifdevopstech/boardgame.git'
            }
        }
        stage('Code Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                  sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                  -Dsonar.java.binaries=.'''
                }
            }
        }
        stage('Quality Gates') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            }
        }
        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                  sh 'mvn deploy'
                } 
            }
        }
        stage('Docker Build & Tag') {
            steps {
               script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                      sh 'docker build -t ari786/boardgame:latest .'
                    }
                }
            }
        }
        stage('Image Scan') {
            steps {
                sh 'trivy image --format table -o image-scan-report.html ari786/boardgame:latest'
            }
        }
        stage('Docker Push') {
            steps {
                script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                      sh 'docker push ari786/boardgame:latest'
                   }
                }
            }
        }
        stage('Deploy the application') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://192.168.1.200:6443') {
                   sh 'kubectl apply -f deployment-service.yaml -n webapps'
                   sh 'kubectl get svc -n webapps'
                }
            }
        }
        
    }
    post {
        success {
            script {
                emailext(
                    subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' was successful.</p>
                             <p>Check console output at <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    to: "${env.RECIPIENTS}"
                )
            }
        }

        failure {
            script {
                emailext(
                    subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed.</p>
                             <p>Check console output at <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    to: "${env.RECIPIENTS}"
                )
            }
        }
    }
}
