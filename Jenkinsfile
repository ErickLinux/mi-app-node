pipeline {
  agent any
  environment {
    PROJECT_NAME    = 'mi-app-node'
    PROJECT_VERSION = "${env.BUILD_NUMBER}"
    DTRACK_URL      = 'http://localhost:8081'
    DTRACK_API_KEY  = credentials('dtrack_api_key')
  }
  stages {
    stage('Verify Tools') {
      steps {
        script {
          echo '🔍 Verificando herramientas instaladas...'
          def pandocVersion = bat(
            script: 'pandoc --version',
            returnStdout: true
          ).trim().split('\n')[0]
          echo "✅ ${pandocVersion}"
          
          def nodeVersion = bat(
            script: 'node --version',
            returnStdout: true
          ).trim()
          echo "✅ Node.js ${nodeVersion}"
          
          def npmVersion = bat(
            script: 'npm --version',
            returnStdout: true
          ).trim()
          echo "✅ npm ${npmVersion}"
        }
      }
    }

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build (npm)') {
      steps {
        bat 'npm ci --no-audit'
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
        // Usando parámetros válidos para el plugin Dependency Track
        dependencyTrackPublisher(
          artifact: 'bom.json',
          projectName: env.PROJECT_NAME,
          projectVersion: env.PROJECT_VERSION,
          autoCreateProjects: true,
          synchronous: true
          // waitForResults y failOnError no son parámetros válidos en esta versión
        )
      }
    }

    stage('Wait for Analysis Completion') {
      steps {
        script {
          // Esperar manualmente para que el análisis se complete
          echo '⏳ Esperando que Dependency Track complete el análisis...'
          sleep time: 60, unit: 'SECONDS' // Esperar 60 segundos
          echo '✅ Análisis debería estar completo'
        }
      }
    }

    stage('Export Findings (via API)') {
      steps {
        script {
          writeFile file: 'export-findings.ps1', text: '''
$apiKey = $env:DTRACK_API_KEY
$dtrackUrl = $env:DTRACK_URL
$projectName = $env:PROJECT_NAME
$projectVersion = $env:PROJECT_VERSION

Write-Host "🔍 Buscando proyecto: $projectName version: $projectVersion"
$lookupUrl = "${dtrackUrl}/api/v1/project/lookup?name=${projectName}&version=${projectVersion}"

try {
    $lookup = Invoke-RestMethod -Headers @{ 'X-Api-Key' = $apiKey } -Uri $lookupUrl -Method Get
    $uuid = $lookup.uuid
    Write-Host "✅ Proyecto encontrado - UUID: $uuid"
    
    $exportUrl = "${dtrackUrl}/api/v1/finding/project/${uuid}/export"
    Invoke-RestMethod -Headers @{ 'X-Api-Key' = $apiKey } -OutFile findings.json -Uri $exportUrl -Method Get
    Write-Host "✅ findings.json descargado exitosamente"
    
} catch {
    Write-Host "❌ Error: $($_.Exception.Message)"
    if ($_.ErrorDetails.Message) {
        Write-Host "Detalles: $($_.ErrorDetails.Message)"
    }
    exit 1
}
'''
          bat 'powershell -ExecutionPolicy Bypass -File export-findings.ps1'
          bat 'if not exist findings.json (echo ERROR: findings.json no encontrado && exit 1)'
        }
      }
    }

    stage('Generate Report with Pandoc') {
      steps {
        script {
          writeFile file: 'generate-report.ps1', text: '''
# Script para generar reporte con Pandoc
$findingsFile = "findings.json"
if (-not (Test-Path $findingsFile)) {
    Write-Host "❌ ERROR: findings.json no encontrado"
    exit 1
}

Write-Host "📊 Procesando findings.json..."
$f = Get-Content $findingsFile -Raw | ConvertFrom-Json
$proj = $f.project

# Crear contenido Markdown
$md = "# Informe de Analisis de Vulnerabilidades\n\n"
$md += "**Proyecto:** $($proj.name)\n\n"
$md += "**Version:** $($proj.version)\n\n"
$md += "**Fecha del Analisis:** $(Get-Date -Format \"yyyy-MM-dd HH:mm:ss\")\n\n"
$md += "**Herramienta:** Dependency Track + Jenkins + Pandoc\n\n"

# Resumen por severidad
$md += "## Resumen por Nivel de Severidad\n\n"
$counts = @{}
foreach ($i in $f.findings) {
    $sev = if ($i.vulnerability.severity) { $i.vulnerability.severity } else { \"UNASSIGNED\" }
    if ($counts.ContainsKey($sev)) { $counts[$sev]++ } else { $counts[$sev] = 1 }
}

$severityOrder = @(\"CRITICAL\",\"HIGH\",\"MEDIUM\",\"LOW\",\"UNASSIGNED\")
foreach ($k in $severityOrder) {
    if ($counts.ContainsKey($k)) { 
        $md += \"- **$k**: $($counts[$k]) vulnerabilidades\n\" 
    }
}

# Tabla de hallazgos detallados
$md += \"\n## Detalle de Hallazgos\n\n\"
$md += \"| Componente | Vulnerabilidad | Severidad | CVSS | URL |\n\"
$md += \"|------------|----------------|-----------|------|-----|\n\"

foreach ($i in $f.findings) {
    $comp = $i.component
    $v = $i.vulnerability
    
    # Escapar pipes para markdown
    $name = \"$($comp.name)@$($comp.version)\".Replace(\"|\", \"&#124;\")
    $vuln = \"$($v.source)-$($v.vulnId)\".Replace(\"|\", \"&#124;\")
    $sev = if ($v.severity) { $v.severity } else { \"UNASSIGNED\" }
    
    $score = if ($v.cvssV3BaseScore) { $v.cvssV3BaseScore } 
             elseif ($v.cvssV2BaseScore) { $v.cvssV2BaseScore } 
             else { \"N/A\" }
    
    $url = if ($v.url) { $v.url } else { \"N/A\" }
    
    $md += \"| $name | $vuln | $sev | $score | $url |\n\"
}

# Recomendaciones
$md += \"\n## Recomendaciones Generales\n\n\"
$md += \"- **Priorizar** la remediacion de vulnerabilidades CRITICAL y HIGH\n\"
$md += \"- **Actualizar** dependencias a las versiones mas recientes\n\"
$md += \"- **Revisar** advisories oficiales de cada componente\n\"
$md += \"- **Considerar** componentes alternativos si las vulnerabilidades son criticas\n\"
$md += \"- **Monitorear** continuamente las dependencias del proyecto\n\"

# Guardar markdown
Set-Content -Path \"reporte.md\" -Value $md -Encoding UTF8
Write-Host \"✅ reporte.md creado exitosamente\"

# Generar PDF con Pandoc
try {
    Write-Host \"📄 Generando PDF con Pandoc...\"
    pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex
    Write-Host \"✅ reporte.pdf generado exitosamente\"
} catch {
    Write-Host \"❌ Error generando PDF: $($_.Exception.Message)\"
}

# Generar HTML tambien
try {
    Write-Host \"🌐 Generando HTML...\"
    pandoc reporte.md -o reporte.html
    Write-Host \"✅ reporte.html generado exitosamente\"
} catch {
    Write-Host \"❌ Error generando HTML: $($_.Exception.Message)\"
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
      echo '✅ Pipeline ejecutado exitosamente!'
      echo '📦 Artefactos disponibles para descarga'
    }
    failure {
      echo '❌ Pipeline falló. Revisar logs para detalles.'
    }
  }
}