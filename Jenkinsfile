pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'petclinic'
        DOCKER_TAG = "${BUILD_NUMBER}"
        MAVEN_DOCKER_IMAGE = 'maven:3.9.5-eclipse-temurin-17'
        MAVEN_OPTS = '-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true'
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
                sh 'mvn clean package -DskipTests -Dhttp.keepAlive=false -Dmaven.wagon.http.pool=false -Dmaven.wagon.httpconnectionManager.ttlSeconds=120 -Denforcer.skip=true'
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
                sh 'mvn test -Dhttp.keepAlive=false -Dmaven.wagon.http.pool=false -Dmaven.wagon.httpconnectionManager.ttlSeconds=120 -Denforcer.skip=true'
            }
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
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
            steps {
                script {
                    docker.withRegistry('https://your-docker-registry-url', 'docker-credentials-id') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
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
