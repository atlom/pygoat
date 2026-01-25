pipeline{
    agent{
        docker{
            image 'python:3.11-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
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
        stage('git-leaks-scan'){
            steps{
                sh '''
                    set -e
                    apk add --no-cache docker-cli


                    docker run --rm \
                        -v "$PWD:/repo" \
                        -w /repo \
                        zricethezav/gitleaks:latest \
                        dir /repo \
                        --redact \
                        --exit-code 1 \
                        --report-format json \
                        --report-path /repo/gitleaks.json

                    test gitleaks.json
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks.json', fingerprint: true
                }
            }
        }
    }
}