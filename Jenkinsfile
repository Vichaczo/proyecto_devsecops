pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Preparando entorno virtual e instalando dependencias...'
                sh 'python3 -m venv venv'
                sh './venv/bin/pip install -r requirements.txt'
            }
        }

        stage('Test') {
            steps {
                echo 'Ejecutando verificación básica de sintaxis...'
                sh './venv/bin/python -m py_compile app.py'
            }
        }

        stage('Analyze') {
            steps {
                echo 'Ejecutando análisis estático con SonarQube...'
                withSonarQubeEnv('SonarQube') {
                    sh "${tool 'sonar-scanner'}/bin/sonar-scanner -Dsonar.projectKey=proyecto-devsecops -Dsonar.sources=. -Dsonar.exclusions=venv/**"
                }
            }
        }

        stage('Security Test - Dependency-Check') {
            steps {
                echo 'Analizando vulnerabilidades en dependencias con OWASP Dependency-Check...'
                dependencyCheck additionalArguments: '-o "./" -s "./" -f "ALL" --prettyPrint --scan requirements.txt --disableNodeAudit --disableYarnAudit --nvdValidForHours 24', odcInstallation: 'dependency-check'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        } // <-- Aquí faltaba esta llave de cierre de los steps y la stage

        stage('Deploy') {
            steps {
                echo 'Desplegando la aplicación en entorno de prueba...'
                sh 'nohup ./venv/bin/python app.py > flask.log 2>&1 &'
                sh 'sleep 3'
            }
        }

        stage('Security Test - OWASP ZAP') {
            steps {
                echo 'Ejecutando escaneo dinámico (DAST) con OWASP ZAP...'
                sh '''
                    docker run --rm --network="host" \
                    -v $(pwd):/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t http://localhost:5000/hello?name=test -r zap_report.html || true
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno y deteniendo servidor de pruebas...'
            sh 'pkill -f "python app.py" || true'
            archiveArtifacts artifacts: 'zap_report.html, dependency-check-report.*', allowEmptyArchive: true
        }
    }
}
