pipeline {
    agent any
    
    tools{
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment{
        SCANNER_HOME= tool "sonar-scanner"
    }
    

    stages {
        stage('Git checkout') {
            steps {
               git branch: 'main', changelog: false, poll: false, url: 'https://github.com/atlurisatheesh/mission-app.git'
            }
        }
        
        stage('compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Triivy scan files') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube ') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=mission1 -Dsonar.projectName=mission1 \
                    -Dsonar.java.binaries=.'''
                
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Deploy Artifacts nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-token', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Build & tag image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t atluri1993/mission:latest ."
                    }
                }
            }
        }
        
         stage('Triivy scan image') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html atluri1993/mission:latest"
            }
        }
        
        stage('Publish tag image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker push atluri1993/mission:latest "
                    }
                }
            }
        }
        
        stage('Deploy image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker run -d -p 8083:8080 atluri1993/mission:latest "
                    }
                }
            }
        }
    }
}
