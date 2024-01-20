Deployment of Amazon clone app using Terraform and jenkins ci-cd

Pre-requisite
aws account
basics of terraform and jenkins

Let's do it
before do anything →
open your terminal and make a separate folder for amazon →mkdir amazon
cd amazon
clone the github repo
git clone https://github.com/Aakibgithuber/Amazon-app-Deployment-using-terraform-and-jenkins.git

Completion steps →
Step 1 → Setup Terraform and configure aws on your local machine
Step 2 → Building a simple Infrastructure from code using terraform
Step 3 → Setup Sonarqube and jenkins
Step 4 → ci-cd pipeline
Step5 → Monitering via Prmotheus and grafana

jenkins pipeline file is below 

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
                git branch: 'main', url: 'https://github.com/Aakibgithuber/Amazon-app-Deployment-using-terraform-and-jenkins.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Amazon \
                    -Dsonar.projectKey=Amazon '''
                }
            }
        }
         stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins' 
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
                       sh "docker build -t amazon-clone ."
                       sh "docker tag amazon-clone aakibkhan1212/amazon-clone:latest "
                       sh "docker push aakibkhan1212/amazon-clone:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image aakibkhan1212/amazon-clone:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name amazon-clone -p 3000:3000 aakibkhan1212/amazon-clone:latest'
            }
        }
    }
}
                
