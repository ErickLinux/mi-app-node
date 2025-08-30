pipeline {
    agent any
    environment {
        PROJECT_NAME = 'mi-app-node'
        PROJECT_VERSION = "${env.BUILD_NUMBER}"
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

        stage('Create Security Report') {
            steps {
                script {
                    // Crear un reporte básico sin Dependency Track
                    bat """
                    echo # Reporte de Analisis de Seguridad > reporte.md
                    echo ## Proyecto: ${env.PROJECT_NAME} >> reporte.md
                    echo ## Version: ${env.PROJECT_VERSION} >> reporte.md
                    echo ## Build: ${env.BUILD_NUMBER} >> reporte.md
                    echo ## Fecha: %DATE% %TIME% >> reporte.md
                    echo. >> reporte.md
                    echo ### Resultados del Analisis >> reporte.md
                    echo - ✅ BOM generado exitosamente >> reporte.md
                    echo - 📋 Lista de dependencias creada >> reporte.md
                    echo - 🔍 Listo para analisis de vulnerabilidades >> reporte.md
                    echo. >> reporte.md
                    echo ### Próximos Pasos >> reporte.md
                    echo 1. Revisar el BOM en Dependency Track >> reporte.md
                    echo 2. Identificar vulnerabilidades >> reporte.md
                    echo 3. Actualizar dependencias >> reporte.md
                    echo 4. Ejecutar análisis periódicos >> reporte.md
                    """
                    
                    // Generar PDF
                    bat 'pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex'
                    
                    // Generar HTML
                    bat 'pandoc reporte.md -o reporte.html'
                }
            }
        }

        stage('Manual Dependency Track Upload') {
            steps {
                script {
                    echo '📋 Para completar el análisis:'
                    echo '1. Abre http://localhost:8080 en tu navegador'
                    echo '2. Ve a Projects → Create Project'
                    echo '3. Nombre: mi-app-node'
                    echo '4. Sube manualmente el archivo bom.json'
                    echo '5. Revisa los resultados de vulnerabilidades'
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
            echo '📦 Artefactos generados:'
            echo '   - bom.json (Bill of Materials)'
            echo '   - reporte.md (Reporte Markdown)'
            echo '   - reporte.html (Reporte HTML)'
            echo '   - reporte.pdf (Reporte PDF)'
            echo '🌐 Abre http://localhost:8080 para analizar vulnerabilidades'
        }
        failure {
            echo '❌ Pipeline falló'
            echo '💡 Revisa los logs para detalles'
        }
    }
}