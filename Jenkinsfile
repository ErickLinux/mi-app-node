pipeline {
    agent any

    environment {
        DTRACK_URL      = 'http://localhost:8080'   // Dependency-Track URL
        DTRACK_API_KEY  = credentials('dtrack-api-key') // Jenkins secret ID
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
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
                dependencyTrackPublisher artifacts: [[artifact: 'bom.json', source: 'bom.json']]
            }
        }

        stage('Export Findings (via API)') {
            steps {
                bat '''
                powershell -Command ^
                $f = Get-Content findings.json -Raw | ConvertFrom-Json; ^
                $proj = $f.project; ^
                $fecha = Get-Date -Format o; ^
                $md = "# Informe de Vulnerabilidades - Dependency-Track`n`n" + `
                      "**Proyecto:** ${proj.name}`n" + `
                      "**Versión:** ${proj.version}`n" + `
                      "**Fecha:** $fecha`n`n## Resumen por severidad`n"; ^
                $counts = @{}; ^
                foreach ($i in $f.findings) { ^
                    $sev = ($i.vulnerability.severity); ^
                    if (-not $sev) { $sev='UNASSIGNED'}; ^
                    if ($counts.ContainsKey($sev)) { $counts[$sev]++ } else { $counts[$sev]=1 } ^
                }; ^
                foreach ($k in @('CRITICAL','HIGH','MEDIUM','LOW','UNASSIGNED')) { ^
                    if ($counts.ContainsKey($k)) { $md += "- $($k): $($counts[$k])`n" } ^
                }; ^
                Set-Content -Path reporte.md -Value $md -Encoding UTF8; ^
                Write-Output 'reporte.md creado';
                '''
            }
        }

        stage('Generate Report (Markdown → HTML/PDF)') {
            steps {
                bat '''
                powershell -Command ^
                pandoc reporte.md -o reporte.html; ^
                pandoc reporte.md -o reporte.pdf;
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.md, *.html, *.pdf', allowEmptyArchive: true
        }
    }
}
