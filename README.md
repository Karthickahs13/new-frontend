# new-frontend
pipeline {
    agent any
    environment {
        // Set up environment variables for SonarQube, Nexus, and Docker Hub credentials
        SONAR_HOST_URL = 'http://sonarqube.example.com'
        SONAR_AUTH_TOKEN = credentials('sonar-auth-token')
        NEXUS_URL = 'http://nexus.example.com/repository/npm-repo'
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_IMAGE_NAME = 'my-angular-app'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-org/your-angular-app.git'
            }
        }
        
        stage('Build Angular Application') {
            steps {
                sh 'npm install'
                sh 'npm run build --prod'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'npm run sonar'
                }
            }
        }
        
        stage('Upload Tarball to Nexus') {
            steps {
                sh 'tar -czvf angular-app.tar.gz dist/*'
                nexusArtifactUploader artifacts: [
                    [artifactId: 'angular-app', classifier: '', file: 'angular-app.tar.gz', type: 'tar.gz']
                ], credentialsId: 'nexus-credentials', groupId: 'com.example', nexusUrl: 'http://nexus.example.com', nexusVersion: 'nexus3', protocol: 'http', repository: 'npm-repo', version: '1.0.0'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE_NAME:latest .'
            }
        }
        
        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        sh "docker push $DOCKER_IMAGE_NAME:latest"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean up the workspace after each build
        }
    }
}

