pipeline {
    agent any

    stages {
        stage('Print docker-compose.yml') {
            steps {
                script {
                    def composeFile = readFile 'docker-compose.yml'
                    echo "Contents of docker-compose.yml:"
                    echo composeFile
                }
            }
        }
    }
}
