pipeline{
    agent{
        docker{
            image 'python:3.11-alpine'
        }
    }
    environment{
        DTRACK_URL = "http://host.docker.internal:8081" //No lo tengo en la misma red, pero eso se usa esta url.
        DTRACK_PROJECT_NAME = "pygoat"
        DD_URL = "http://host.docker.internal:8085" 
        DD_ENGAGEMENT_ID = "1" 
    }
    stages{
        stage('bandit-scan'){
            steps {
                script{
                    sh 'pip install bandit'
                    int rc = sh(
                        script: 'bandit -r . -f json -o bandit.json',
                        returnStatus: true
                    )
                    //Marca como UNSTABLE en caso de que el bandit encuentre vulnerabilidades.
                    if (rc != 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Bandit encontró hallazgos (exit code ${rc}). El pipeline continúa."
                    } else {
                        echo "Bandit sin hallazgos."
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit.json', fingerprint: true
                }
            }
        }
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
                    apk add --no-cache curl tar git

                    GITLEAKS_VERSION="8.18.4"
                    curl -sL "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" | tar -xz
                    chmod +x gitleaks

                    ./gitleaks detect \
                    --source . \
                    --redact \
                    --exit-code 0 \
                    --report-format json \
                    --report-path gitleaks.json
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks.json', fingerprint: true
                }
            }
        }
        stage('DefectDojo-upload'){
            steps{
                withCredentials([string(credentialsId: 'DefectDojo', variable: 'DD_TOKEN')]){
                    sh '''
                        apk add --no-cache curl ca-certificates

                         # Bandit
                        test -f bandit.json
                        curl -sS -X POST "$DD_URL/api/v2/reimport-scan/" \
                            -H "Authorization: Token $DD_TOKEN" \
                            -F "engagement=$DD_ENGAGEMENT_ID" \
                            -F "scan_type=Bandit" \
                            -F "test_title=bandit" \
                            -F "file=@bandit.json" >/dev/null

                        # Gitleaks
                        test -f gitleaks.json
                        curl -sS -X POST "$DD_URL/api/v2/reimport-scan/" \
                            -H "Authorization: Token $DD_TOKEN" \
                            -F "engagement=$DD_ENGAGEMENT_ID" \
                            -F "scan_type=Gitleaks" \
                            -F "test_title=gitleaks" \
                            -F "file=@gitleaks.json" >/dev/null
                    '''
                }
            }
        }
    }
}