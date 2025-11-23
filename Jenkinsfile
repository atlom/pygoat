pipeline{
    agent any

    stages{
        stage('Dependency-Track'){
            steps {
                script{
                    sh 'docker compose pull'
                    sh 'docker compose up -d'
                }
            }
        }
    }
}