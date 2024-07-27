pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'desync/petc'
        // DOKERHUB_REPO = 'desync/petc'
        DOCKER_TAG = "${BUILD_NUMBER}"
        MAVEN_DOCKER_IMAGE = 'maven:3.9.5-eclipse-temurin-17'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            agent {
                docker {
                    image "${MAVEN_DOCKER_IMAGE}"
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh '''
                    mvn clean package -DskipTests \
                    -Dcheckstyle.skip=true \
                    -Dspring-javaformat.skip=true \
                    -Denforcer.skip=true
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image "${MAVEN_DOCKER_IMAGE}"
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh '''
                    mvn test \
                    -Dcheckstyle.skip=true \
                    -Dspring-javaformat.skip=true \
                    -Denforcer.skip=true
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: '''
                        FROM eclipse-temurin:17-jre-alpine
                        VOLUME /tmp
                        COPY target/*.jar app.jar
                        ENTRYPOINT ["java","-jar","/app.jar"]
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Push Docker Image') {
            environment {
                DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
            }
            steps {
                script {
                    sh "echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
            post {
                always {
                    sh "docker logout"
                }
            }
        }

        stage('Deploy') {
            steps {
                sh "docker run -d -p 8080:8080 ${DOCKER_IMAGE}:${DOCKER_TAG}"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
