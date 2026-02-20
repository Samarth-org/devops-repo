pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_CREDENTIALS = credentials("dockerhub-creds")
        IMAGE_NAME = "${DOCKER_CREDENTIALS_USR}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        // Comment this out if you haven't created the secret in Jenkins yet
        // SONAR_TOKEN = credentials('sonarcloud-token')
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
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-creds') {
                        def docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker_image.push()
                        docker_image.push("latest")
                    }
                }
            }
        }
        stage("Trivy Vulnerability Scan") {
            steps {
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} --severity HIGH,CRITICAL --format table"
            }
        }
        stage("Deploy for DAST") {
            steps {
                script {
                    sh "docker rm -f test-app || true"
                    sh "docker run -d -p 8082:8080 --name test-app ${IMAGE_NAME}:${IMAGE_TAG}"
                    sleep 15 
                }
            }
        }
        stage("DAST - Security Scan") {
            steps {
                // Notice: docker run for test-app is removed here because it's already running
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
            steps { cleanWs() }
        }
    }
}