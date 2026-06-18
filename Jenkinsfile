pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        APP_NAME        = "boardgame"
        DOCKER_IMAGE    = "tanuj7777777/boardgame"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        SONAR_HOME      = tool 'sonar-scanner'
        RECIPIENT_EMAIL = "tanujnimkar.cloud@gmail.com"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-creds',
                    url: 'https://github.com/TanUjNimkar/end-to-end-devops-cicd-pipeline.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh '''
                   trivy fs \
                   --format table \
                   --severity HIGH,CRITICAL \
                   --output trivy-fs-report.html \
                   .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                       mvn sonar:sonar \
                       -Dsonar.projectName=BoardGame \
                       -Dsonar.projectKey=BoardGame \
                       -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings',
                          jdk: 'jdk17',
                          maven: 'maven3',
                          traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-creds',
                                       url: 'https://index.docker.io/v1/') {
                        sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} -t ${DOCKER_IMAGE}:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh """
                   trivy image \
                   --format table \
                   --severity HIGH,CRITICAL \
                   --output trivy-image-report.html \
                   ${DOCKER_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-creds',
                                       url: 'https://index.docker.io/v1/') {
                        sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8s-creds',
                               clusterName: '',
                               caCertificate: '',
                               contextName: '',
                               namespace: 'webapps',
                               restrictKubeConfigAccess: false,
                               serverUrl: '') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8s-creds',
                               clusterName: '',
                               caCertificate: '',
                               contextName: '',
                               namespace: 'webapps',
                               restrictKubeConfigAccess: false,
                               serverUrl: '') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def status = currentBuild.currentResult
                def emoji  = status == 'SUCCESS' ? '✅' : '❌'
                def color  = status == 'SUCCESS' ? '#28a745' : '#dc3545'

                emailext(
                    subject: "${emoji} ${env.JOB_NAME} | Build #${env.BUILD_NUMBER} | ${status}",
                    mimeType: 'text/html',
                    to: "${RECIPIENT_EMAIL}",
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html',
                    body: """
                        <html>
                        <body style="font-family: Arial; padding: 20px;">
                        <h2 style="color:${color};">${emoji} Build ${status}</h2>
                        <table border="1" cellpadding="8" style="border-collapse:collapse; width:60%;">
                            <tr><td><b>Job Name</b></td><td>${env.JOB_NAME}</td></tr>
                            <tr><td><b>Build Number</b></td><td>#${env.BUILD_NUMBER}</td></tr>
                            <tr><td><b>Status</b></td><td style="color:${color};"><b>${status}</b></td></tr>
                            <tr><td><b>Docker Image</b></td><td>${DOCKER_IMAGE}:${IMAGE_TAG}</td></tr>
                            <tr><td><b>Build URL</b></td><td><a href="${env.BUILD_URL}">Click Here</a></td></tr>
                        </table>
                        <br/>
                        <p style="color:grey; font-size:12px;">
                            Automated email from Jenkins — Tanuj Nimkar
                        </p>
                        </body>
                        </html>
                    """
                )
            }

            sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true"
            sh "docker rmi ${DOCKER_IMAGE}:latest || true"
            sh "docker image prune -f || true"
        }

        success {
            echo 'Pipeline completed successfully'
        }

        failure {
            echo 'Pipeline failed please check the logs'
        }
    }
}
