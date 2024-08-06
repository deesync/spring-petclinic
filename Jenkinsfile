pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'desync/petc'
        DOCKER_TAG = "${BUILD_NUMBER}"
        MAVEN_DOCKER_IMAGE = 'maven:3.9.5-eclipse-temurin-17'

        SONAR_PROJECT_KEY = 'spring-petclinic'
        SONAR_SERVER_URL = 'http://192.168.56.71:9000'
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

        stage('Analyse & Build') {
            failFast false
            parallel {
                stage('SonarQube Analysis') {
                    agent {
                        docker {
                            image 'sonarsource/sonar-scanner-cli:latest'
                            args '-v $HOME/.sonar:/home/sonar/.sonar -v ${WORKSPACE}:/usr/src'
                        }
                    }
                    environment {
                        SONAR_TOKEN = credentials('jenkins-sonar')
                    }
                    steps {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_SERVER_URL} \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.projectBaseDir=/usr/src
                        '''
                    }
                    post {
                        failure {
                            echo 'SonarQube analysis failed! Ignore to continue the pipeline...'
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
                    post {
                        success {
                            archiveArtifacts(artifacts: '**/target/*.jar', allowEmptyArchive: true)
                        }
                    }
                }
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
                    
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                    
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
            environment {
                CONTAINER_NAME = 'petclinic'
                DEPLOY_SERVER_URL = '192.168.56.20'
                DEPLOY_DIR = '/opt/deploy'
                DEPLOY_CREDS = credentials('deployment-server')
                DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
            }
            steps {
                script {
                    // Create a docker-compose.yml file
                    writeFile file: 'docker-compose.yml', text: """
                        services:
                            petclinic:
                                image: ${DOCKER_IMAGE}:latest
                                ports:
                                - "8080:8080"
                                restart: always
                    """

                    // Copy the docker-compose file to the server and deploy
                    sshagent(credentials: ['deployment-server']) {
                        sh "scp docker-compose.yml ${DEPLOY_CREDS_USR}@${DEPLOY_SERVER_URL}:${DEPLOY_DIR}"
                        
                        // Login to Docker Hub, check existing container, remove if exists, pull new image, and deploy
                        sh """
                            ssh ${DEPLOY_CREDS_USR}@${DEPLOY_SERVER_URL} '
                                echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin &&
                                cd ${DEPLOY_DIR} &&
                                if docker ps -a | grep -q ${CONTAINER_NAME}; then
                                    echo "Container ${CONTAINER_NAME} exists. Removing it."
                                    docker stop ${CONTAINER_NAME} || true
                                    docker rm ${CONTAINER_NAME} || true
                                else
                                    echo "Container ${CONTAINER_NAME} does not exist. Proceeding with deployment."
                                fi &&
                                docker compose pull && 
                                docker compose up -d --force-recreate &&
                                docker logout
                            '
                        """
                        
                    }
                }
            }
        }
    }

    post {
        success {
            cleanWs()
        }
    }
}
