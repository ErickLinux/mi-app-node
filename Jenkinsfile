pipeline {
    agent any
    environment {
        PROJECT_NAME = 'mi-app-node'
        PROJECT_VERSION = "${env.BUILD_NUMBER}"
        DTRACK_URL = 'http://localhost:8080'
        DTRACK_API_KEY = credentials('dtrack_api_key')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }

        stage('Generate BOM') {
            steps {
                bat 'npx @cyclonedx/cyclonedx-npm --output-file bom.json'
                bat 'if exist bom.json (echo ✅ BOM generado) else (echo ❌ Error generando BOM && exit 1)'
            }
        }

        stage('Upload to Dependency Track') {
            steps {
                script {
                    echo "📤 Subiendo BOM a Dependency-Track..."
                    dependencyTrackPublisher(
                        artifact: 'bom.json',
                        projectName: env.PROJECT_NAME,
                        projectVersion: env.PJECT_VERSION,
                        autoCreateProjects: true
                    )
                }
            }
        }

        stage('Generate Simple Report') {
            steps {
                script {
                    // Crear un reporte simple de demostración
                    bat '''
                    echo # Reporte de Seguridad > reporte.md
                    echo ## Proyecto: %PROJECT_NAME% >> reporte.md
                    echo ## Build: %BUILD_NUMBER% >> reporte.md
                    echo ## Fecha: %DATE% %TIME% >> reporte.md
                    echo ### Este es un reporte de demostración >> reporte.md
                    echo - Analisis completado con Dependency Track >> reporte.md
                    echo - Reporte generado con Pandoc >> reporte.md
                    '''
                    
                    // Convertir a PDF
                    bat 'pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex || echo ⚠️ PDF no generado, continuando...'
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'bom.json, reporte.md, reporte.pdf', fingerprint: true
            cleanWs()
        }
        success {
            echo '✅ Pipeline completado exitosamente'
        }
        failure {
            echo '❌ Pipeline falló - revisar logs'
        }
    }
}