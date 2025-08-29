pipeline {
    agent any
    environment {
        DTRACK_URL = 'http://localhost:8080'
    }
    stages {
        stage('Build SBOM') {
            steps {
                bat 'npx @cyclonedx/cyclonedx-npm --output-file bom.json --output-format json'
            }
        }
        stage('Upload to Dependency-Track') {
            steps {
                dependencyTrackPublisher(
                    artifact: 'bom.json',
                    projectName: 'mi-app-node',
                    projectVersion: '1.0.0',
                    synchronous: true,
                    credentialsId: 'dtrack-apikey'
                )
            }
        }
    }
}
