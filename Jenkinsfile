pipeline {
    agent any
    
    stages {
        stage('Build & Test') {
            steps {
                echo 'Preparando entorno e instalando dependencias...'
                sh 'python3 -m venv venv'
                sh './venv/bin/pip install -r requirements.txt'
                // Aquí podrías correr pytest si tuvieras pruebas unitarias
            }
        }
        
        stage('Analyze (SonarQube)') {
            steps {
                echo 'Ejecutando análisis de código con SonarQube...'
                // Nota: Requiere configurar el SonarQube Scanner previamente en Jenkins
                // def scannerHome = tool 'SonarQubeScanner'
                // sh "${scannerHome}/bin/sonar-scanner ..."
            }
        }

        stage('Security Test (Dependency-Check)') {
            steps {
                echo 'Analizando vulnerabilidades en dependencias...'
                // dependencyCheck additionalArguments: '', odcInstallation: 'Default'
            }
        }
        
        stage('Deploy (Local Test)') {
            steps {
                echo 'Desplegando la aplicación en entorno de prueba...'
                // Ejemplo para levantar Flask en segundo plano para las pruebas de ZAP
                sh 'nohup ./venv/bin/python app.py &> flask.log & '
            }
        }
        
        stage('Security Test (OWASP ZAP)') {
            steps {
                echo 'Ejecutando escaneo dinámico con OWASP ZAP...'
                // Aquí invocarías el contenedor de ZAP o el plugin para apuntar a http://localhost:5000/hello
            }
        }
    }
    
    post {
        always {
            echo 'Limpiando entorno o finalizando pipeline...'
            sh 'pkill -f app.py || true' // Detiene Flask al terminar
        }
    }
}
