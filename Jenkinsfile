pipeline {
  agent any
  environment {
    PROJECT_NAME    = 'mi-app-node'
    PROJECT_VERSION = "${env.BUILD_NUMBER}"
    DTRACK_URL      = 'http://localhost:8081'  // ¡Cambié a 8081! Dependency Track usa 8081 por defecto
    DTRACK_API_KEY  = credentials('dtrack_api_key')
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build (npm)') {
      steps {
        bat 'npm ci'
      }
    }

    stage('Generate SBOM (CycloneDX)') {
      steps {
        bat 'npx @cyclonedx/cyclonedx-npm --output-file bom.json --output-format json'
        bat 'if not exist bom.json (echo ERROR: bom.json no encontrado && exit 1)'
      }
    }

    stage('Upload SBOM to Dependency-Track') {
      steps {
        dependencyTrackPublisher(
          artifact: 'bom.json',
          projectName: env.PROJECT_NAME,
          projectVersion: env.PROJECT_VERSION,
          autoCreateProjects: true,
          synchronous: true,
          waitForResults: true,
          failOnError: false
        )
      }
    }

    stage('Export Findings (via API)') {
      steps {
        script {
          // Crear archivo PowerShell separado y ejecutarlo
          writeFile file: 'export-findings.ps1', text: '''
$apiKey = $env:DTRACK_API_KEY
$dtrackUrl = $env:DTRACK_URL
$projectName = $env:PROJECT_NAME
$projectVersion = $env:PROJECT_VERSION

Write-Host "Buscando proyecto: $projectName version: $projectVersion"

# Lookup del proyecto
$lookupUrl = "${dtrackUrl}/api/v1/project/lookup?name=${projectName}&version=${projectVersion}"
Write-Host "URL de lookup: $lookupUrl"

try {
    $lookup = Invoke-RestMethod -Headers @{ 'X-Api-Key' = $apiKey } -Uri $lookupUrl -Method Get
    Write-Host "Proyecto encontrado: $($lookup.name) - $($lookup.version)"
    
    $uuid = $lookup.uuid
    Write-Host "UUID del proyecto: $uuid"
    
    # Exportar findings
    $exportUrl = "${dtrackUrl}/api/v1/finding/project/${uuid}/export"
    Write-Host "Exportando findings desde: $exportUrl"
    
    Invoke-RestMethod -Headers @{ 'X-Api-Key' = $apiKey } -OutFile findings.json -Uri $exportUrl -Method Get
    Write-Host "findings.json descargado exitosamente"
    
} catch {
    Write-Host "Error: $($_.Exception.Message)"
    Write-Host "Response: $($_.ErrorDetails.Message)"
    exit 1
}
'''
          bat 'powershell -ExecutionPolicy Bypass -File export-findings.ps1'
          bat 'if not exist findings.json (echo ERROR: findings.json no encontrado && exit 1)'
        }
      }
    }

    stage('Generate Report (Markdown → HTML/PDF)') {
      steps {
        script {
          writeFile file: 'generate-report.ps1', text: '''
# Script para generar reporte
$findingsFile = "findings.json"
if (-not (Test-Path $findingsFile)) {
    Write-Host "ERROR: findings.json no encontrado"
    exit 1
}

$f = Get-Content $findingsFile -Raw | ConvertFrom-Json
$proj = $f.project

$md = "# Informe de Vulnerabilidades - Dependency-Track`n`n"
$md += "**Proyecto:** $($proj.name)`n`n"
$md += "**Versión:** $($proj.version)`n`n"
$md += "**Fecha:** $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')`n`n"
$md += "## Resumen por severidad`n`n"

# Contar vulnerabilidades por severidad
$counts = @{}
foreach ($i in $f.findings) {
    $sev = if ($i.vulnerability.severity) { $i.vulnerability.severity } else { 'UNASSIGNED' }
    if ($counts.ContainsKey($sev)) { 
        $counts[$sev]++ 
    } else { 
        $counts[$sev] = 1 
    }
}

# Ordenar por severidad
$severityOrder = @('CRITICAL','HIGH','MEDIUM','LOW','UNASSIGNED')
foreach ($k in $severityOrder) {
    if ($counts.ContainsKey($k)) { 
        $md += "- $k`: $($counts[$k])`n" 
    }
}

$md += "`n## Hallazgos`n`n"
$md += "| Componente | Vulnerabilidad | Severidad | CVSS | URL |`n"
$md += "|------------|----------------|-----------|------|-----|`n"

foreach ($i in $f.findings) {
    $comp = $i.component
    $v = $i.vulnerability
    
    $name = "$($comp.name)@$($comp.version)" -replace '\|','\\|'
    $vuln = "$($v.source)-$($v.vulnId)" -replace '\|','\\|'
    $sev = if ($v.severity) { $v.severity } else { 'UNASSIGNED' }
    
    $score = if ($v.cvssV3BaseScore) { $v.cvssV3BaseScore } 
             elseif ($v.cvssV2BaseScore) { $v.cvssV2BaseScore } 
             else { 'N/A' }
    
    $url = if ($v.url) { $v.url } else { 'N/A' }
    
    $md += "| $name | $vuln | $sev | $score | $url |`n"
}

$md += "`n## Recomendaciones generales`n`n"
$md += "- Priorizar la remediación de vulnerabilidades CRITICAL y HIGH`n"
$md += "- Actualizar las dependencias a las versiones más recientes`n"
$md += "- Revisar los advisories y notas de cada componente vulnerables`n"
$md += "- Considerar componentes alternativos si las vulnerabilidades son críticas`n"

Set-Content -Path "reporte.md" -Value $md -Encoding UTF8
Write-Host "reporte.md creado exitosamente"

# Convertir a HTML y PDF si Pandoc está disponible
try {
    # HTML
    pandoc reporte.md -o reporte.html
    Write-Host "reporte.html generado"
} catch {
    Write-Host "No se pudo generar HTML: $($_.Exception.Message)"
}

try {
    # PDF
    pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex
    Write-Host "reporte.pdf generado"
} catch {
    Write-Host "No se pudo generar PDF: $($_.Exception.Message)"
}
'''
          bat 'powershell -ExecutionPolicy Bypass -File generate-report.ps1'
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'bom.json, findings.json, reporte.md, reporte.html, reporte.pdf', fingerprint: true
      cleanWs()
    }
    success {
      echo 'Pipeline ejecutado exitosamente!'
    }
    failure {
      echo 'Pipeline falló. Revisar logs para detalles.'
    }
  }
}