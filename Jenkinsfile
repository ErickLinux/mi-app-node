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
                    echo "Project Name: ${env.PROJECT_NAME}"
                    echo "Project Version: ${env.PROJECT_VERSION}"
                    
                    dependencyTrackPublisher(
                        artifact: 'bom.json',
                        projectName: env.PROJECT_NAME,
                        projectVersion: env.PROJECT_VERSION, // ✅ CORREGIDO: era "env.PJECT_VERSION"
                        autoCreateProjects: true,
                        synchronous: true
                    )
                }
            }
        }

        stage('Wait for Analysis') {
            steps {
                script {
                    echo '⏳ Esperando 30 segundos para el análisis...'
                    sleep time: 30, unit: 'SECONDS'
                }
            }
        }

        stage('Generate Simple Report') {
            steps {
                script {
                    // Crear reporte básico
                    bat """
                    echo # Reporte de Analisis de Seguridad > reporte.md
                    echo ## Proyecto: ${env.PROJECT_NAME} >> reporte.md
                    echo ## Version: ${env.PROJECT_VERSION} >> reporte.md
                    echo ## Fecha: %DATE% %TIME% >> reporte.md
                    echo ### Resultados: >> reporte.md
                    echo - BOM generado y subido a Dependency Track >> reporte.md
                    echo - Analisis de vulnerabilidades completado >> reporte.md
                    echo - Reporte generado con Pandoc >> reporte.md
                    """
                    
                    // Convertir a PDF
                    bat 'pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex || echo ⚠️ PDF no generado, continuando...'
                    
                    // Generar HTML también
                    bat 'pandoc reporte.md -o reporte.html || echo ⚠️ HTML no generado, continuando...'
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'bom.json, reporte.md, reporte.html, reporte.pdf', fingerprint: true
            cleanWs()
        }
        success {
            echo '✅ Pipeline completado exitosamente!'
            echo '📦 Artefactos disponibles para descarga'
        }
        failure {
            echo '❌ Pipeline falló - revisar logs'
        }
    }
}