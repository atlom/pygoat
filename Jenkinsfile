pipeline{
    agent{
        docker{
            image 'python:3.11-alpine'
        }
    }
    environment{
        DTRACK_URL = "http://host.docker.internal:8081" //No lo tengo en la misma red, pero eso se usa esta url
        DTRACK_PROJECT_NAME = "pygoat"
    }
    stages{
        // stage('bandit-scan'){
        //     steps {
        //         script{
        //             sh 'pip install bandit'
        //             sh 'bandit -r .'
        //         }
        //     }
        // }
        stage('dependency-track-scan') {
            steps {
                withCredentials([string(credentialsId: 'DepTrack', variable: 'DTRACK_API_KEY')]) {
                    sh '''
                        test -f requirements.txt

                        pip install --no-cache-dir cyclonedx-bom
                        cyclonedx-py requirements \
                            -i requirements.txt \
                            -o bom.json \
                            --output-format json

                        test -s bom.json

                        apk add --no-cache curl ca-certificates

                        curl -sS -X POST "$DTRACK_URL/api/v1/bom" \
                        -H "X-Api-Key: $DTRACK_API_KEY" \
                        -H "Content-Type: multipart/form-data" \
                        -F "projectName=$DTRACK_PROJECT_NAME" \
                        -F "autoCreate=true" \
                        -F "bom=@bom.json"

                    '''
                }
                
            }
        }
    }
}