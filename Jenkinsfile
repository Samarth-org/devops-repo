pipeline {
    agent any
    
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage('Verify Environment') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
            }
        }

        stage('Check Docker') {
            steps {
                sh 'which docker'
                sh 'docker --version'
            }
        }

        stage("Build Application") {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    // Explicitly fetching username to avoid the "null string" validation error
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        
                        def fullImageName = "${DOCKER_USER}/${APP_NAME}"
                        
                        docker.withRegistry('', 'dockerhub-creds') {
                            def docker_image = docker.build("${fullImageName}:${IMAGE_TAG}")
                            docker_image.push()
                            docker_image.push("latest")
                        }
                    }
                }
            }
        }

        stage("Trivy Vulnerability Scan") {
            steps {
                script {
                    withCredentials([usernameVariable(credentialsId: 'dockerhub-creds', variable: 'DOCKER_USER')]) {
                        sh """
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} \
                            --severity HIGH,CRITICAL --format table
                        """
                    }
                }
            }
        }

        stage("Deploy for DAST") {
            steps {
                script {
                    withCredentials([usernameVariable(credentialsId: 'dockerhub-creds', variable: 'DOCKER_USER')]) {
                        sh "docker rm -f test-app || true"
                        sh "docker run -d -p 8082:8080 --name test-app ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                        echo "Waiting for application to stabilize..."
                        sleep 15 
                    }
                }
            }
        }

        stage("DAST - Security Scan") {
            steps {
                // OWASP ZAP scan against the container running on 8082
                sh 'docker run --rm --network="host" owasp/zap2docker-stable zap-baseline.py -t http://localhost:8082 -r zap-report.html || true'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
                    sh "docker rm -f test-app || true"
                }
            }
        }

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }
    }
}