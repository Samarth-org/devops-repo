@Library('my-shared-library') _

pipeline {
    agent {
        docker {
            // This image comes pre-loaded with Maven, JDK 17, and Docker CLI
            image 'trion/maven-docker:3.9.6-jdk17'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    options {
        // Prevents the "Permission Denied" errors during checkout
        skipDefaultCheckout(true)
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SLACK_CHANNEL = '#test-notify'
    }

    stages {
        stage('Cleanup & Checkout') {
            steps {
                deleteDir()
                checkout scm
                script {
                    // Quick verification that tools are present
                    sh 'mvn -version'
                    sh 'docker --version'
                    notify("Environment Ready")
                }
            }
        }

        stage("Build Application") {
            steps {
                // Since you have a pom.xml at the root, this will build your project
                sh 'mvn clean package -DskipTests'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        def fullImage = "${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                        
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker build -t ${fullImage} ."
                        sh "docker push ${fullImage}"
                        sh "docker logout"
                        
                        notify("Build #${env.BUILD_NUMBER} Pushed to Hub")
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", color: "good", 
                message: "*BUILD SUCCESS* \nJob: ${env.JOB_NAME} \nBuild: #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", color: "danger", 
                message: "*BUILD FAILED* \nJob: ${env.JOB_NAME} \nBuild: #${env.BUILD_NUMBER}")
        }
    }
}