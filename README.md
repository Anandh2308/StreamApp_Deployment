# Netflix Clone - CI/CD Pipeline Deployment on Kubernetes

This project demonstrates the end-to-end CI/CD pipeline for deploying a **Netflix clone** using Jenkins, Docker, Kubernetes, SonarQube, Trivy, and other tools. The application is containerized and deployed on a Kubernetes cluster (one master, two slaves), with integrated monitoring via Prometheus, Node Exporter, and Grafana.

## Project Overview

This project automates the process of building, analyzing, scanning, and deploying a Netflix clone application. The pipeline is managed by Jenkins and incorporates various DevOps tools for code quality checks, security scans, and container orchestration.

### Technologies Used

- **Git & GitHub**: For version control.
- **Jenkins**: CI/CD tool to automate the build and deployment pipeline.
- **Node.js & JDK**: Runtime environments for the application.
- **SonarQube**: Static code analysis and quality gates.
- **OWASP Dependency Check**: To ensure secure dependencies.
- **Trivy**: Vulnerability scanning for Docker images.
- **Docker**: For containerization of the Netflix clone application.
- **Kubernetes**: For container orchestration (1 master, 2 slaves).
- **Prometheus & Node Exporter**: For monitoring system and application metrics.
- **Grafana**: Visualization of metrics from Prometheus.
- **Mail Notifications**: Real-time email updates on pipeline status.

## CI/CD Pipeline Workflow

1. **Tool Verification**: 
   - Jenkins checks the installed versions of JDK and Node.js.
   - Verifies the correct Git branch for deployment.
   
2. **Code Quality Analysis**: 
   - Jenkins runs **SonarQube** to analyze code quality and enforces quality gates.
   
3. **Dependency Management & Build**: 
   - Dependencies are installed and managed.
   - Application is built using the specified tools.

4. **Docker Build & Push**: 
   - Jenkins builds a Docker image for the Netflix clone and pushes it to a Docker registry.

5. **Security Scanning**: 
   - **Trivy** and **OWASP Dependency Check** are run to scan for vulnerabilities in dependencies and the Docker image.
   
6. **Kubernetes Deployment**: 
   - Jenkins deploys the application to the Kubernetes cluster.
   - Deployment is verified through health checks.

7. **Monitoring Setup**: 
   - **Prometheus** (with **Node Exporter**) monitors the application and system performance.
   - **Grafana** visualizes real-time metrics and dashboards.

8. **Post-Deployment Notification**: 
   - Email notifications are sent at the end of the pipeline to update on the deployment status.

## Monitoring & Metrics

- **Prometheus** is set up with **Node Exporter** to monitor system-level metrics (CPU, memory, etc.).
- **Grafana** provides a visualization of these metrics through custom dashboards, enabling better insights into application performance.

## How to Run the Project

1. Clone the repository from GitHub:
   ```bash
   git clone https://github.com/anandh2308/streamapp_deployment.git

2. Set up Jenkins and configure the pipeline script:
   ```bash
   pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/anandh2308/streamapp_deployment'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=StreamApp \
                    -Dsonar.projectKey=StreamApp '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=addkey -t streamapp ."
                       sh "docker tag netflix anandh2308/streamapp:23.08 "
                       sh "docker push anandh2308/streamapp:23.08 "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image anandh2308/streamapp:23.08 > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 anandh2308/streamapp:23.08'
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: 'Kubernetes', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }
         stage('Verify the Deployments'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: 'Kubernetes', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl get pods -n webapps'
                                sh 'kubectl get svc -n webapps'
                        }   
                    }
                }
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'xxx@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}



