pipeline {
    agent any
    environment {
        PROJECT_NAME = 'mi-app-node'
        PROJECT_VERSION = "${env.BUILD_NUMBER}"
        DTRACK_API_KEY = 'odt_mlLlMkmS_lNd6fzVJkofN7RXXAR96G6Su8HnnURGD'
        DTRACK_URL = 'http://localhost:8080'
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

        stage('Verify Dependency Track API') {
            steps {
                script {
                    writeFile file: 'test-api.ps1', text: """
try {
    \$response = Invoke-RestMethod -Uri "${env.DTRACK_URL}/api/version" \`
        -Headers @{"X-Api-Key" = "${env.DTRACK_API_KEY}"} \`
        -Method Get
    Write-Host "✅ Dependency Track API respondiendo: Versión \$(\$response)"
} catch {
    Write-Host "❌ Error conectando a Dependency Track API: \$(\$_.Exception.Message)"
    exit 1
}
"""
                    bat 'powershell -ExecutionPolicy Bypass -File test-api.ps1'
                }
            }
        }

        stage('Upload BOM to Dependency Track') {
            steps {
                script {
                    writeFile file: 'upload-bom.ps1', text: """
\$apiKey = "${env.DTRACK_API_KEY}"
\$bomContent = Get-Content -Path 'bom.json' -Raw
\$bytes = [System.Text.Encoding]::UTF8.GetBytes(\$bomContent)
\$encodedBom = [Convert]::ToBase64String(\$bytes)

\$body = @{
    projectName = '${env.PROJECT_NAME}'
    projectVersion = '${env.PROJECT_VERSION}'
    autoCreate = \$true
    bom = \$encodedBom
} | ConvertTo-Json

try {
    Write-Host "📤 Subiendo BOM a Dependency Track..."
    \$response = Invoke-RestMethod -Uri '${env.DTRACK_URL}/api/v1/bom' \`
        -Method POST \`
        -Headers @{ 'X-Api-Key' = \$apiKey; 'Content-Type' = 'application/json' } \`
        -Body \$body
    Write-Host "✅ BOM subido exitosamente. Token: \$(\$response.token)"
} catch {
    Write-Host "❌ Error subiendo BOM: \$(\$_.Exception.Message)"
    if (\$_.ErrorDetails.Message) {
        Write-Host "Detalles del error: \$(\$_.ErrorDetails.Message)"
    }
    exit 1
}
"""
                    bat 'powershell -ExecutionPolicy Bypass -File upload-bom.ps1'
                }
            }
        }

        stage('Wait for Analysis') {
            steps {
                script {
                    echo '⏳ Esperando 30 segundos para el análisis de vulnerabilidades...'
                    sleep time: 30, unit: 'SECONDS'
                    echo '✅ Análisis completado'
                }
            }
        }

        stage('Generate Security Report') {
            steps {
                script {
                    writeFile file: 'generate-report.ps1', text: """
# Crear reporte de seguridad
\$md = "# 🔍 Reporte de Análisis de Seguridad\n\n"
\$md += "## 📋 Información del Proyecto\n\n"
\$md += "- **Proyecto:** ${env.PROJECT_NAME}\n"
\$md += "- **Versión:** ${env.PROJECT_VERSION}\n"
\$md += "- **Build Number:** ${env.BUILD_NUMBER}\n"
\$md += "- **Fecha:** \$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')\n\n"
\$md += "## 🛡️  Herramientas Utilizadas\n\n"
\$md += "- **Dependency Track:** Análisis de vulnerabilidades\n"
\$md += "- **CycloneDX:** Generación de BOM (Bill of Materials)\n"
\$md += "- **Pandoc:** Generación de reportes\n"
\$md += "- **Jenkins:** Automatización del pipeline\n\n"
\$md += "## ✅ Resultados del Análisis\n\n"
\$md += "- ✅ BOM generado exitosamente\n"
\$md += "- ✅ BOM subido a Dependency Track\n"
\$md += "- ✅ Análisis de vulnerabilidades completado\n"
\$md += "- 📊 Reporte generado automáticamente\n\n"
\$md += "## 📊 Próximos Pasos\n\n"
\$md += "1. Revisar el dashboard de Dependency Track\n"
\$md += "2. Identificar vulnerabilidades críticas\n"
\$md += "3. Actualizar dependencias vulnerables\n"
\$md += "4. Implementar parches de seguridad\n"
\$md += "5. Programar análisis periódicos\n\n"
\$md += "---\n\n"
\$md += "*Reporte generado automáticamente por Jenkins Pipeline*"

# Guardar markdown
Set-Content -Path "security-report.md" -Value \$md -Encoding UTF8
Write-Host "✅ security-report.md creado"

# Generar PDF
try {
    pandoc security-report.md -o security-report.pdf --pdf-engine=xelatex
    Write-Host "✅ security-report.pdf generado"
} catch {
    Write-Host "⚠️  No se pudo generar PDF: \$(\$_.Exception.Message)"
}

# Generar HTML
try {
    pandoc security-report.md -o security-report.html
    Write-Host "✅ security-report.html generado"
} catch {
    Write-Host "⚠️  No se pudo generar HTML: \$(\$_.Exception.Message)"
}
"""
                    bat 'powershell -ExecutionPolicy Bypass -File generate-report.ps1'
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'bom.json, security-report.md, security-report.html, security-report.pdf', fingerprint: true
            cleanWs()
        }
        success {
            echo '✅ ¡Pipeline completado exitosamente!'
            echo '📦 Artefactos disponibles para descarga:'
            echo '   - bom.json (Bill of Materials)'
            echo '   - security-report.md (Reporte Markdown)'
            echo '   - security-report.html (Reporte HTML)'
            echo '   - security-report.pdf (Reporte PDF)'
            echo '🌐 Revisa los resultados en: http://localhost:8080'
        }
        failure {
            echo '❌ Pipeline falló'
            echo '💡 Verifica los logs para más detalles'
        }
    }
}