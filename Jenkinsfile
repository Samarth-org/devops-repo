// pipeline {
//     agent any
    
//     tools {
//         jdk 'Java17'
//         maven 'Maven3'
//     }

//     environment {
//         APP_NAME = "register-app-pipeline"
//         RELEASE = "1.0.0"
//         IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
//     }

//     stages {
//         stage('Verify Environment') {
//             steps {
//                 sh 'java -version'
//                 sh 'mvn -version'
//             }
//         }

//         stage('Check Docker') {
//             steps {
//                 sh 'which docker'
//                 sh 'docker --version'
//             }
//         }

//         stage("Build Application") {
//             steps {
//                 sh 'mvn clean package -DskipTests'
//             }
//         }

//         stage("Build & Push Docker Image") {
//             steps {
//                 script {
//                     // This block securely retrieves your Docker Hub credentials
//                     withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        
//                         // Define the full image path
//                         def imagePath = "${DOCKER_USER}/${APP_NAME}"
//                         def fullImage = "${imagePath}:${IMAGE_TAG}"
                        
//                         echo "Logging into Docker Hub..."
//                         sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        
//                         echo "Building image: ${fullImage}"
//                         // The '.' tells Docker the Dockerfile is in the current root directory
//                         sh "docker build -t ${fullImage} ."
                        
//                         echo "Tagging and Pushing..."
//                         sh "docker tag ${fullImage} ${imagePath}:latest"
//                         sh "docker push ${fullImage}"
//                         sh "docker push ${imagePath}:latest"
                        
//                         echo "Cleaning up local login..."
//                         sh "docker logout"
//                     }
//                 }
//             }
//         }

//         stage("Trivy Vulnerability Scan") {
//             steps {
//                 script {
//                     withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'UNUSED_PASS', usernameVariable: 'DOCKER_USER')]) {
//                         sh """
//                             docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
//                             aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} \
//                             --severity HIGH,CRITICAL --format table
//                         """
//                     }
//                 }
//             }
//         }

//         stage("Cleanup Workspace") {
//             steps {
//                 cleanWs()
//             }
//         }
//     }
// }


@Library('my-shared-library') _

pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
            // Keep -u root to allow Docker CLI installation inside the container
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SLACK_CHANNEL = '#test-notify' // Un-commented and active
    }

    stages {
        stage('Initialize & Clean') {
            steps {
                // Ensure the workspace is clean of root-owned files from previous runs
                deleteDir() 
                script {
                    // Install Docker CLI
                    sh 'curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh'
                    notify("Environment Verified") 
                }
            }
        }

        stage("Build Application") {
            steps {
                sh 'mvn clean package -DskipTests'
                notify("Build Complete")
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        def imagePath = "${DOCKER_USER}/${APP_NAME}"
                        def fullImage = "${imagePath}:${IMAGE_TAG}"
                        
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker build -t ${fullImage} ."
                        sh "docker push ${fullImage}"
                        sh "docker logout"
                        
                        notify("Image Pushed to Docker Hub")
                    }
                }
            }
        }

        stage("Trivy Vulnerability Scan") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'UNUSED_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} --severity HIGH,CRITICAL --format table"
                    }
                }
            }
        }
    }

    post {
        always {
            // This ensures workspace cleanup happens even on failure
            cleanWs()
        }
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