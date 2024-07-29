pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'desync/petc'
        DOCKER_TAG = "${BUILD_NUMBER}"
        MAVEN_DOCKER_IMAGE = 'maven:3.9.5-eclipse-temurin-17'
        SONAR_PROJECT_KEY = 'spring-petclinic'
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='spring-petclinic' \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    reuseNode true
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
                    sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                    sh 'docker push ${DOCKER_IMAGE}:${DOCKER_TAG}'
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
