@Library('my-shared-library') _

pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
            // Using root is fine for installing tools, but we must clean up
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    options {
        // This tells Jenkins to delete the workspace BEFORE the git checkout starts
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
                // 1. Manually clean the workspace inside the container (as root)
                deleteDir()
                // 2. Now manually trigger the checkout
                checkout scm
                script {
                    sh 'curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh'
                    notify("Environment Ready")
                }
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
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        def fullImage = "${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}"
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker build -t ${fullImage} ."
                        sh "docker push ${fullImage}"
                        sh "docker logout"
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", color: "good", 
                message: "*BUILD SUCCESS* \nJob: ${env.JOB_NAME} \nBuild: #${env.BUILD_NUMBER} \nURL: ${env.BUILD_URL}")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", color: "danger", 
                message: "*BUILD FAILED* \nJob: ${env.JOB_NAME} \nBuild: #${env.BUILD_NUMBER} \nLogs: ${env.BUILD_URL}")
        }
    }
}