
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

                dependencyCheck(
                    odcInstallation: 'dependency-check',
                    additionalArguments: '-o "./" -s "./" -f "ALL" --prettyPrint --scan requirements.txt --disableNodeAudit --disableYarnAudit --disableCentral --nvdApiDelay 10000 --nvdMaxRetryCount 10 --nvdValidForHours 24'
                )

                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando la aplicación en entorno de prueba...'

                sh '''
                    nohup ./venv/bin/python app.py > flask.log 2>&1 < /dev/null &

                    for i in $(seq 1 15); do
                        if curl -fsS "http://localhost:5000/hello?name=test" > /dev/null; then
                            echo "Flask está disponible en el puerto 5000."
                            exit 0
                        fi

                        echo "Esperando a que Flask inicie..."
                        sleep 2
                    done

                    echo "ERROR: Flask no pudo iniciar."
                    cat flask.log
                    exit 1
                '''
            }
        }

        stage('Security Test - OWASP ZAP') {
            steps {
                echo 'Ejecutando escaneo dinámico (DAST) con OWASP ZAP...'

                sh '''
                    docker run --rm --network="host" \
                    -v "$(pwd):/zap/wrk/:rw" \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                    -t "http://localhost:5000/hello?name=test" \
                    -r zap_report.html \
                    -I
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno y deteniendo servidor de pruebas...'

            sh 'pkill -f "python app.py" || true'

            archiveArtifacts(
                artifacts: 'zap_report.html, dependency-check-report.*, flask.log',
                allowEmptyArchive: true
            )
        }
    }
}
