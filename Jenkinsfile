pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_USER = 'naresh9163'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/Naresh0591/Hotstar-devops.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Hotstar \
                        -Dsonar.projectKey=Hotstar'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: true }
            }
        }
        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }
        stage('TRIVY FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: 'https://index.docker.io/v1/') {
                        withCredentials([string(credentialsId: 'tmdb-api-key', variable: '68f46e27dfbb53cb1f47418ffb3fb8a1')]) {
                            sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t hotstar hotstar/"
                            sh "docker tag netflix ${DOCKERHUB_USER}/hotstar:${BUILD_NUMBER}"
                            sh "docker push ${DOCKERHUB_USER}/hotstar:${BUILD_NUMBER}"
                        }
                    }
                }
            }
        }
        stage('TRIVY Image Scan') {
            steps { sh "trivy image ${DOCKERHUB_USER}/hotstar:${BUILD_NUMBER} > trivyimage.txt" }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            export AWS_DEFAULT_REGION=us-east-1
                            aws eks update-kubeconfig --region us-east-1 --name cloudhotstar
                            sed -i 's|naresh9163/hotstar:latest|naresh9163/hotstar:${BUILD_NUMBER}|g' deployment.yml
                            kubectl apply -f deployment.yml
                            kubectl get pods
                            kubectl get svc
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline execution complete.'
        }
        success {
            echo 'Pipeline succeeded! Application deployed to EKS.'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
            emailext(
                subject: "FAILED: Pipeline '${env.JOB_NAME}' [${env.BUILD_NUMBER}]",
                body: "Build failed. Check console output at: ${env.BUILD_URL}",
                to: 'nareshtullibilli666@gmail.com'
            )
        }
    }
}
