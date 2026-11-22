
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Preparando entorno virtual e instalando dependencias...'

                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Ejecutando verificación de sintaxis e importación de Flask...'

                sh '''
                    ./venv/bin/python -m py_compile app.py
                    ./venv/bin/python -c "from flask import Flask; print('Flask importado correctamente')"
                '''
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
                    nvdCredentialsId: 'nvd-api-key',
                    additionalArguments: '-o "./" -s "./" -f "ALL" --prettyPrint --scan requirements.txt --disableNodeAudit --disableYarnAudit --disableCentral --nvdApiDelay 3500 --nvdMaxRetryCount 10 --nvdValidForHours 24'
                )

                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando la aplicación Flask en entorno de prueba...'

                sh '''
                    JENKINS_NODE_COOKIE=dontKillMe \
                    nohup ./venv/bin/python app.py > flask.log 2>&1 < /dev/null &

                    echo $! > flask.pid

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
                    mkdir -p zap-reports
                    chmod 777 zap-reports

                    docker run --rm --network="host" \
                        -v "$(pwd)/zap-reports:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t "http://localhost:5000/hello?name=test" \
                        -r zap_report.html \
                        -I

                    test -s zap-reports/zap_report.html
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno y deteniendo servidor de pruebas...'

            sh '''
                if [ -f flask.pid ]; then
                    PID=$(cat flask.pid)

                    if [ -n "$PID" ] && kill -0 "$PID" 2>/dev/null; then
                        kill "$PID" 2>/dev/null || true
                    fi

                    rm -f flask.pid
                fi
            '''

            archiveArtifacts(
                artifacts: 'zap-reports/zap_report.html, dependency-check-report.*, flask.log',
                allowEmptyArchive: true
            )
        }
    }
}
