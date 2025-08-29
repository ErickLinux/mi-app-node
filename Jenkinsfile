pipeline {
  agent any
  environment {
    PROJECT_NAME    = 'mi-app-node'               
    PROJECT_VERSION = "${env.BUILD_NUMBER}"
    DTRACK_URL      = 'http://localhost:8080'   // Dependency-Track corre en 8080
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
          projectName: PROJECT_NAME,
          projectVersion: PROJECT_VERSION,
          autoCreateProjects: true,
          synchronous: true
        )
      }
    }

    stage('Export Findings (via API)') {
      steps {
        bat """
powershell -Command ^
  $env:APIKEY='${DTRACK_API_KEY}'; ^
  $lookup = Invoke-RestMethod -Headers @{ 'X-Api-Key' = $env:APIKEY } -Uri '${DTRACK_URL}/api/v1/project/lookup?name=${PROJECT_NAME}&version=${PROJECT_VERSION}'; ^
  if (-not $lookup) { Write-Error 'Project not found'; exit 1 } ^
  $uuid = $lookup.uuid; ^
  Invoke-RestMethod -Headers @{ 'X-Api-Key' = $env:APIKEY } -OutFile findings.json -Uri \"${DTRACK_URL}/api/v1/finding/project/$uuid/export\"; ^
  Write-Output 'findings.json descargado';
"""
        bat 'if not exist findings.json (echo ERROR: findings.json no encontrado && exit 1)'
      }
    }

    stage('Generate Report (Markdown → HTML/PDF)') {
      steps {
        bat """
powershell -Command ^
  $f = Get-Content findings.json -Raw | ConvertFrom-Json; ^
  $proj = $f.project; ^
  $md = \"# Informe de Vulnerabilidades - Dependency-Track`n`n**Proyecto:** ${proj.name}  `n**Versión:** ${proj.version}  `n**Fecha:** $(Get-Date -Format o)`n`n## Resumen por severidad`n\" ; ^
  $counts = @{}; ^
  foreach ($i in $f.findings) { $sev = ($i.vulnerability.severity) ; if (-not $sev) { $sev = 'UNASSIGNED' }; if ($counts.ContainsKey($sev)) { $counts[$sev]++ } else { $counts[$sev]=1 } } ; ^
  foreach ($k in @('CRITICAL','HIGH','MEDIUM','LOW','UNASSIGNED')) { if ($counts.ContainsKey($k)) { $md += \"- $k: $($counts[$k])`n\" } } ; ^
  $md += \"`n## Hallazgos`n| Componente | Vulnerabilidad | Severidad | CVSS | URL |`n|---|---|---|---|---|`n\" ; ^
  foreach ($i in $f.findings) { ^
    $comp = $i.component; $v = $i.vulnerability; ^
    $name = \"$($comp.name)@$($comp.version)\" -replace '\\|','\\|'; ^
    $vuln = \"$($v.source)-$($v.vulnId)\" -replace '\\|','\\|'; ^
    $sev = ($v.severity) -replace '\\|','\\|'; ^
    $score = $v.cvssV3BaseScore; if (-not $score) { $score = $v.cvssV2BaseScore } ; ^
    $url = $v.url; ^
    $md += \"| $name | $vuln | $sev | $score | $url |`n\" ; ^
  } ; ^
  $md += \"`n## Recomendaciones generales`n- Priorizar CRITICAL/HIGH y actualizar dependencias. `n- Revisar advisory y notas del componente.`n\" ; ^
  Set-Content -Path reporte.md -Value $md -Encoding UTF8; ^
  Write-Output 'reporte.md creado';
"""
        bat 'pandoc reporte.md -o reporte.html || echo Pandoc HTML falló'
        bat 'pandoc reporte.md -o reporte.pdf --pdf-engine=xelatex || echo PDF no generado (falta LaTeX)'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'bom.json, findings.json, reporte.*', fingerprint: true
    }
  }
}
