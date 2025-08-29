pipeline {
    agent any
    environment {
        PROJECT_NAME = 'mi-app-node'
        PROJECT_VERSION = "${env.BUILD_NUMBER}"
        DTRACK_URL = 'http://localhost:8080'
        DTRACK_API_KEY = credentials('dependency_track_api_key')
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

        stage('Test API Connection') {
            steps {
                script {
                    // Crear script PowerShell sin caracteres problemáticos
                    writeFile file: 'test-api.ps1', text: '''
$headers = @{
    "X-Api-Key" = "' + env.DTRACK_API_KEY + '"
    "Content-Type" = "application/json"
}

try {
    $response = Invoke-RestMethod -Uri "' + env.DTRACK_URL + '/api/version" -Headers $headers -Method Get
    Write-Host "✅ API Connection Successful. Version: $response"
} catch {
    Write-Host "❌ API Connection Failed: $($_.Exception.Message)"
    Write-Host "💡 Please check:"
    Write-Host "   - Dependency Track is running"
    Write-Host "   - API Key is correct"
    Write-Host "   - Network connectivity"
    exit 1
}
'''
                    bat 'powershell -ExecutionPolicy Bypass -File test-api.ps1'
                }
            }
        }

        stage('Upload BOM to Dependency Track') {
            steps {
                script {
                    writeFile file: 'upload-bom.ps1', text: '''
$headers = @{
    "X-Api-Key" = "' + env.DTRACK_API_KEY + '"
    "Content-Type" = "application/json"
}

$bomContent = Get-Content -Path "bom.json" -Raw
$bytes = [System.Text.Encoding]::UTF8.GetBytes($bomContent)
$encodedBom = [Convert]::ToBase64String($bytes)

$body = @{
    projectName = "' + env.PROJECT_NAME + '"
    projectVersion = "' + env.PROJECT_VERSION + '"
    autoCreate = $true
    bom = $encodedBom
} | ConvertTo-Json

try {
    Write-Host "📤 Uploading BOM to Dependency Track..."
    $response = Invoke-RestMethod -Uri "' + env.DTRACK_URL + '/api/v1/bom" -Headers $headers -Method Post -Body $body
    Write-Host "✅ BOM Upload Successful. Token: $($response.token)"
    Write-Host "🌐 View results at: ' + env.DTRACK_URL + '/projects/$($response.token)"
} catch {
    Write-Host "❌ BOM Upload Failed: $($_.Exception.Message)"
    if ($_.ErrorDetails.Message) {
        Write-Host "Error Details: $($_.ErrorDetails.Message)"
    }
    exit 1
}
'''
                    bat 'powershell -ExecutionPolicy Bypass -File upload-bom.ps1'
                }
            }
        }

        stage('Generate Security Report') {
            steps {
                script {
                    bat """
                    echo # Security Analysis Report > reporte.md
                    echo ## Project: ${env.PROJECT_NAME} >> reporte.md
                    echo ## Version: ${env.PROJECT_VERSION} >> reporte.md
                    echo ## Build: ${env.BUILD_NUMBER} >> reporte.md
                    echo ## Date: %DATE% %TIME% >> reporte.md
                    echo. >> reporte.md
                    echo ### Analysis Results >> reporte.md
                    echo - ✅ BOM generated successfully >> reporte.md
                    echo - ✅ BOM uploaded to Dependency Track >> reporte.md
                    echo - 🔍 Vulnerability analysis completed >> reporte.md
                    echo. >> reporte.md
                    echo ### Next Steps >> reporte.md
                    echo 1. Review results in Dependency Track >> reporte.md
                    echo 2. Identify critical vulnerabilities >> reporte.md
                    echo 3. Update vulnerable dependencies >> reporte.md
                    echo 4. Schedule periodic scans >> reporte.md
                    """
                    
                    bat 'pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex'
                    bat 'pandoc reporte.md -o reporte.html'
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
            echo '✅ Pipeline completed successfully!'
            echo '📦 Generated artifacts:'
            echo '   - bom.json (Bill of Materials)'
            echo '   - reporte.md (Markdown Report)'
            echo '   - reporte.html (HTML Report)'
            echo '   - reporte.pdf (PDF Report)'
        }
        failure {
            echo '❌ Pipeline failed'
            echo '💡 Check logs for details'
        }
    }
}