pipeline{
    agent any

    stages{
        stage('Dependency-Track'){
            steps {
                script{
                    sh 'curl -LO https://dependencytrack.org/docker-compose.yml'
                    sh 'docker compose up -d'
                }
            }
        }
    }
}