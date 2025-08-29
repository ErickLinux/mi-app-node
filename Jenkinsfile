pipeline {
    agent any

    environment {
        DTRACK_URL = 'http://localhost:8080'          // URL de Dependency-Track
        DTRACK_API_KEY = credentials('dtrack-apikey') // ID de la credencial con tu API Key
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
                dependencyTrackPublisher(
                    artifact: 'bom.json',
                    projectName: 'mi-app-node',
                    projectVersion: '1.0.0',
                    serverUrl: "${DTRACK_URL}",
                    apiKey: "${DTRACK_API_KEY}",
                    synchronous: true
                )
            }
        }

        stage('Post Actions') {
            steps {
                archiveArtifacts artifacts: '**/*', fingerprint: true
            }
        }
    }
}
