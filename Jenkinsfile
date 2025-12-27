pipeline{
    agent any

    stages{

        stage('install-safety'){
            agent{
                docker{
                    image 'python:3.11-alpine'
                }
            }
            steps {
                script{
                    sh 'pip install safety'
                    sh 'safety --version'
                    sh 'safety scan'
                }
            }
        }

        // stage('Docker-Compose'){
        //     steps {
        //         script{
        //             sh 'docker compose pull'
        //             sh 'docker compose up -d'
        //         }
        //     }
        // }

        // stage('dependencyTrackPublisher'){
        //     agent {
        //         docker{
        //             image 'maven:sapmachine'
        //             args '-u root'
        //         }
        //     }
        //     steps{
        //         script{
        //             sh 'mvn org.cyclonedx-maven-plugin:2.7.0:makeAgregateBom'
        //         }
        //         withCredentials([string(credentialsId: 'DepTrack', variable: 'API_KEY')]) {
        //             dependencyTrackPublisher artifact: 'target/bom.xml', projectName: 'my-project', projectVersion: 'my-version', synchronous: true, dependencyTrackApiKey: API_KEY, projectProperties: [tags: ['tag1', 'tag2'], swidTagId: 'my swid tag', group: 'my group', parentId: 'parent-uuid']
        //         }
        //     }
        // }
    }
}