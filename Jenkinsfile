pipeline{
    agent any

    stages{
        stage('Dependency-Track'){
            step {
                sh 'curl -LO https://dependencytrack.org/docker-compose.yml'
                sh 'docker compose up -d'
            }
        }
    }
}