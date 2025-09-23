pipeline {
    // กำหนด Agent ที่จะใช้ (ต้องมี Docker ติดตั้ง)
    agent any

    // กำหนดเครื่องมือที่จำเป็น
    tools {
        maven 'Maven-3.8.5' // ชื่อที่ตั้งใน Jenkins > Global Tool Configuration
    }

    // กำหนด Environment Variables
    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        N8N_WEBHOOK_URL_CREDENTIALS = 'n8n-webhook-url'
        DOCKER_IMAGE_NAME = 'iamsamitdev/springboot-docker-app'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        // Stage 1: ดึงโค้ดล่าสุดจาก Git
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        // Stage 2: รัน Unit Tests และ Integration Tests
        stage('Run Tests') {
            steps {
                echo 'Running tests with Maven...'
                sh 'mvn clean test'
            }
            post {
                always {
                    // บันทึกผลการทดสอบ
                    publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                    // เก็บ test coverage report ถ้ามี
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])
                }
            }
        }

        // Stage 3: Build โปรเจกต์ Spring Boot ด้วย Maven
        stage('Build Application') {
            steps {
                echo 'Building the application with Maven...'
                sh 'mvn package -DskipTests'
            }
        }

        // Stage 4: สร้าง Docker Image
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}", '.')
                }
            }
        }

        // Stage 5: Push Image ไปยัง Docker Hub
        stage('Push to Docker Hub') {
            steps {
                script {
                    // ล็อกอินเข้า Docker Hub โดยใช้ Credentials ที่เตรียมไว้
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS) {
                        echo "Pushing image to Docker Hub..."
                        docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push()
                        
                        // (Optional) Push a 'latest' tag
                        docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
    }

    // Post Actions: ส่วนที่จะทำงานหลัง Stages ทั้งหมดเสร็จสิ้น
    post {
        // ทำงานเมื่อ Pipeline สำเร็จเท่านั้น
        success {
            script {
                echo 'CI process completed successfully. Notifying N8N...'
                // ดึง N8N Webhook URL จาก Credentials
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    // ส่ง POST request ไปยัง N8N พร้อมข้อมูลที่เป็นประโยชน์
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "SUCCESS",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "imageUrl": "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}",
                            "dockerHubUrl": "https://hub.docker.com/r/${DOCKER_IMAGE_NAME}/tags",
                            "testsStatus": "PASSED",
                            "buildUrl": "${BUILD_URL}"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
        // ทำงานเมื่อ Pipeline ล้มเหลว
        failure {
            script {
                echo 'CI process failed. Notifying N8N...'
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "FAILED",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "buildUrl": "${BUILD_URL}",
                            "testsStatus": "FAILED"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
        // ทำงานเมื่อ Tests ล้มเหลวแต่ build สำเร็จ
        unstable {
            script {
                echo 'Tests failed but build succeeded. Notifying N8N...'
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "UNSTABLE",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "buildUrl": "${BUILD_URL}",
                            "testsStatus": "FAILED"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
    }
}