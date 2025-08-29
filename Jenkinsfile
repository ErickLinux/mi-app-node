pipeline {
    agent any

    environment {
        DTRACK_URL = 'http://localhost:8080'         // Dependency-Track URL
        DTRACK_API_KEY = credentials('dtrack-api')   // Jenkins credentials ID for API key
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
                dependencyTrackPublisher artifact: 'bom.json'
            }
        }

        stage('Export Findings (via API)') {
            steps {
                echo 'Skipping Export Findings - configure if needed'
            }
        }

        stage('Generate Report (Markdown → HTML/PDF)') {
            steps {
                echo 'Skipping Report Generation - configure if needed'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/*', allowEmptyArchive: true
        }
    }
}
