pipeline {
    // ใช้ any agent เพื่อหลีกเลี่ยงปัญหา Docker path mounting บน Windows
    agent any

    // กำหนด environment variables
    environment {
        // ใช้ค่าเป็น "credentialsId" ของ Jenkins โดยตรงสำหรับ docker.withRegistry
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-cred'
        DOCKER_REPO = "iamsamitdev/springboot-docker-app"
        APP_NAME = "springboot-docker-app"
    }

    // กำหนด stages ของ Pipeline
    stages {
        // Stage 1: ดึงโค้ดล่าสุดจาก Git
        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        // Stage 2: ติดตั้ง dependencies และรันเทสต์ (รองรับทุก Platform)
        stage('Install & Test') {
            steps {
                script {
                    // ตรวจสอบว่ามี Java/Maven บน host หรือไม่ ถ้ามีใช้ host; ถ้าไม่มีก็ใช้ Docker maven image
                    def hasMaven = false
                    def isWindows = isUnix() ? false : true

                    try {
                        if (isWindows) {
                            bat 'mvn -v'
                        } else {
                            sh 'mvn -v'
                        }
                        hasMaven = true
                        echo "Using Maven installed on ${isWindows ? 'Windows' : 'Unix'}"
                    } catch (Exception e) {
                        echo "Maven not found on host, using Docker"
                        hasMaven = false
                    }

                    if (hasMaven) {
                        if (isWindows) {
                            bat '''
                                mvn -B -ntp clean test
                            '''
                        } else {
                            sh '''
                                mvn -B -ntp clean test
                            '''
                        }
                    } else {
                        // ใช้ Docker run (maven + jdk) เพื่อรัน test ในทุก platform
                        if (isWindows) {
                            bat '''
                                docker run --rm ^
                                -v "%cd%":/workspace ^
                                -w /workspace ^
                                maven:3.9-eclipse-temurin-21 sh -c "mvn -B -ntp clean test"
                            '''
                        } else {
                            sh '''
                                docker run --rm \
                                -v "$(pwd)":/workspace \
                                -w /workspace \
                                maven:3.9-eclipse-temurin-21 sh -c "mvn -B -ntp clean test"
                            '''
                        }
                    }
                }
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

        // Stage 3: Build โปรเจกต์ (ใช้ host maven ถ้ามี หรือ fallback docker)
        stage('Build Application') {
            steps {
                script {
                    def hasMaven = false
                    def isWindows = isUnix() ? false : true
                    try {
                        if (isWindows) { bat 'mvn -v' } else { sh 'mvn -v' }
                        hasMaven = true
                    } catch (Exception e) { hasMaven = false }

                    if (hasMaven) {
                        if (isWindows) {
                            bat 'mvn -B -ntp -DskipTests package'
                        } else {
                            sh 'mvn -B -ntp -DskipTests package'
                        }
                    } else {
                        if (isWindows) {
                            bat '''
                                docker run --rm ^
                                -v "%cd%":/workspace ^
                                -w /workspace ^
                                maven:3.9-eclipse-temurin-21 sh -c "mvn -B -ntp -DskipTests package"
                            '''
                        } else {
                            sh '''
                                docker run --rm \
                                -v "$(pwd)":/workspace \
                                -w /workspace \
                                maven:3.9-eclipse-temurin-21 sh -c "mvn -B -ntp -DskipTests package"
                            '''
                        }
                    }
                }
            }
        }

        // Stage 4: สร้าง Docker Image สำหรับ production
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_REPO}:${BUILD_NUMBER}"
                    docker.build("${DOCKER_REPO}:${BUILD_NUMBER}", ".")
                }
            }
        }

        // Stage 5: Push Image ไปยัง Docker Hub
        stage('Push Docker Image') {
            steps {
                script {
                    // ต้องส่งค่าเป็น credentialsId เท่านั้น ไม่ใช่ค่าที่ mask ของ credentials()
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_HUB_CREDENTIALS_ID) {
                        echo "Pushing image to Docker Hub..."
                        def image = docker.image("${DOCKER_REPO}:${BUILD_NUMBER}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        // Stage 6: Cleanup local Docker cache/images on the agent to save disk space
        stage('Cleanup Docker') {
            steps {
                script {
                    def isWindows = isUnix() ? false : true
                    echo "Cleaning up local Docker images/cache on agent..."
                    if (isWindows) {
                        bat """
                            docker image rm -f ${DOCKER_REPO}:${BUILD_NUMBER} || echo ignore
                            docker image rm -f ${DOCKER_REPO}:latest || echo ignore
                            docker image prune -af -f
                            docker builder prune -af -f
                        """
                    } else {
                        sh """
                            docker image rm -f ${DOCKER_REPO}:${BUILD_NUMBER} || true
                            docker image rm -f ${DOCKER_REPO}:latest || true
                            docker image prune -af -f
                            docker builder prune -af -f
                        """
                    }
                }
            }
        }

        // Stage 7: Deploy latest image to localhost for smoke test
        stage('Deploy Local') {
            steps {
                script {
                    def isWindows = isUnix() ? false : true
                    echo "Deploying container ${APP_NAME} from latest image..."
                    if (isWindows) {
                        bat """
                            docker pull ${DOCKER_REPO}:latest
                            docker stop ${APP_NAME} || echo ignore
                            docker rm ${APP_NAME} || echo ignore
                            docker run -d --name ${APP_NAME} -p 8080:8080 ${DOCKER_REPO}:latest
                            docker ps --filter name=${APP_NAME} --format \"table {{.Names}}\t{{.Image}}\t{{.Status}}\"
                        """
                    } else {
                        sh """
                            docker pull ${DOCKER_REPO}:latest
                            docker stop ${APP_NAME} || true
                            docker rm ${APP_NAME} || true
                            docker run -d --name ${APP_NAME} -p 8080:8080 ${DOCKER_REPO}:latest
                            docker ps --filter name=${APP_NAME} --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
                        """
                    }
                }
            }
            post {
                success {
                    script {
                        def isWindows = isUnix() ? false : true
                        withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
                            if (isWindows) {
                                // ใช้ PowerShell แบบบรรทัดเดียว (ไม่มี caret ^) เพื่อหลีกเลี่ยง error ใน cmd
                                bat '''
                                    powershell -NoProfile -Command "$body = [PSCustomObject]@{ project=$env:JOB_NAME; stage='Deploy Local'; status='success'; build=$env:BUILD_NUMBER; image=($env:DOCKER_REPO + ':latest'); container=$env:APP_NAME; url='http://localhost:8080/'; timestamp=(Get-Date -Format o) }; $json = $body | ConvertTo-Json; Invoke-RestMethod -Uri $env:N8N_WEBHOOK_URL -Method Post -ContentType 'application/json' -Body $json"
                                '''
                            } else {
                                sh """
                                    curl -s -X POST "$N8N_WEBHOOK_URL" \
                                      -H 'Content-Type: application/json' \
                                      -d '{
                                            "project": "${JOB_NAME}",
                                            "stage": "Deploy Local",
                                            "status": "success",
                                            "build": "${BUILD_NUMBER}",
                                            "image": "${DOCKER_REPO}:latest",
                                            "container": "${APP_NAME}",
                                            "url": "http://localhost:8080/"
                                          }'
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    // แจ้งเตือนเมื่อ Pipeline ล้มเหลว (failure)
    post {
        failure {
            script {
                def isWindows = isUnix() ? false : true
                withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
                    if (isWindows) {
                        bat '''
                            powershell -NoProfile -Command "$body = [PSCustomObject]@{ project=$env:JOB_NAME; stage='Pipeline'; status='failed'; build=$env:BUILD_NUMBER; image=($env:DOCKER_REPO + ':latest'); container=$env:APP_NAME; url='http://localhost:8080/'; timestamp=(Get-Date -Format o) }; $json = $body | ConvertTo-Json; Invoke-RestMethod -Uri $env:N8N_WEBHOOK_URL -Method Post -ContentType 'application/json' -Body $json"
                        '''
                    } else {
                        sh """
                            curl -s -X POST "$N8N_WEBHOOK_URL" \
                              -H 'Content-Type: application/json' \
                              -d '{
                                    "project": "${JOB_NAME}",
                                    "stage": "Pipeline",
                                    "status": "failed",
                                    "build": "${BUILD_NUMBER}",
                                    "image": "${DOCKER_REPO}:latest",
                                    "container": "${APP_NAME}",
                                    "url": "http://localhost:8080/"
                                  }'
                        """
                    }
                }
            }
        }
    }
}