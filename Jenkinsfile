pipeline{
    agent{
        docker{
            image 'python:3.11-alpine'
        }
    }
    environment{
        DTRACK_URL = "http://host.docker.internal:8081" //No lo tengo en la misma red, pero eso se usa esta url.
        DTRACK_PROJECT_NAME = "pygoat"
        DTRACK_PROJECT_VERSION = "main"
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
                            -F "scan_type=Bandit Scan" \
                            -F "test_title=bandit" \
                            -F "file=@bandit.json" >/dev/null

                        # Gitleaks
                        test -f gitleaks.json
                        curl -sS -X POST "$DD_URL/api/v2/reimport-scan/" \
                            -H "Authorization: Token $DD_TOKEN" \
                            -F "engagement=$DD_ENGAGEMENT_ID" \
                            -F "scan_type=Gitleaks Scan" \
                            -F "test_title=gitleaks" \
                            -F "file=@gitleaks.json" >/dev/null
                    '''
                }
            }
        }
        // stage('Bandit-security-gate'){
        //     steps{
        //         script{
        //             sh 'pip install bandit'
        //             int highRisk = sh(
        //                 script: 'bandit -r . -iii -f json -o high-risk-bandit.json',
        //                 returnStatus: true
        //             )
        //             if(highRisk > 0){
        //                 error("Security gate failed: vulnerabilidades críticas")
        //             }
        //         }
        //     }
        // } 
        stage('DependencyTrack-security-gate'){
            steps{
                withCredentials([string(credentialsId: 'DepTrack', variable: 'DTRACK_API_KEY')]) {
                    sh sh '''
        set -eu

        apk add --no-cache curl ca-certificates

        echo "== Lookup project UUID =="
        lookup_url="$DTRACK_URL/api/v1/project/lookup?name=$DTRACK_PROJECT_NAME&version=$DTRACK_PROJECT_VERSION"

        # Traemos el UUID (respuesta es JSON del proyecto)
        project_uuid=$(curl -sS -H "X-Api-Key: $DTRACK_API_KEY" "$lookup_url" | python - <<'PY'
import json,sys
d=json.load(sys.stdin)
print(d.get("uuid",""))
PY
        )

        if [ -z "$project_uuid" ]; then
          echo "ERROR: No pude obtener project UUID. Verifica DTRACK_PROJECT_NAME / DTRACK_PROJECT_VERSION."
          exit 2
        fi

        echo "Project UUID: $project_uuid"

        echo "== Esperando análisis (poll) =="
        # Esperamos hasta 5 min a que haya datos consistentes
        # (D-Track analiza async después del upload del BOM)
        max=30
        i=0
        vulns_json="[]"
        while [ $i -lt $max ]; do
          vulns_json=$(curl -sS -H "X-Api-Key: $DTRACK_API_KEY" \
            "$DTRACK_URL/api/v1/vulnerability/project/$project_uuid" || echo "[]")

          # Si devuelve algo que parezca lista, salimos del loop
          echo "$vulns_json" | python - <<'PY' || true
import json,sys
try:
  d=json.load(sys.stdin)
  assert isinstance(d, list)
  print("OK")
except Exception:
  pass
PY
          if echo "$vulns_json" | python - <<'PY'
import json,sys
d=json.load(sys.stdin)
print(1 if isinstance(d,list) else 0)
PY
          | grep -q '^1$'; then
            break
          fi

          i=$((i+1))
          sleep 10
        done

        echo "== Evaluando severidades (CRITICAL/HIGH) =="

        echo "$vulns_json" | python - <<'PY'
import json,sys
v=json.load(sys.stdin) if sys.stdin.readable() else []
# v es una lista de vulnerabilidades, cada item suele traer severity
crit=0; high=0
for item in v:
  sev = (item.get("severity") or "").upper()
  if sev == "CRITICAL": crit += 1
  elif sev == "HIGH": high += 1
print(f"CRITICAL={crit} HIGH={high} TOTAL={len(v)}")
# Gate: falla si hay crit o high
sys.exit(1 if (crit>0 or high>0) else 0)
PY
      '''
                }
            }
        }   
    }
}