pipeline {
    agent any 
    
    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling monorepo code from GitHub...'
                checkout scm
            }
        }
        
        stage('Test & Analyze') {
            // Notice: NO 'steps' block here, just parallel immediately
            parallel {
                
                stage('Spring Boot (Java 21) & SonarQube') {
                    agent {
                        docker { 
                            image 'maven:3.9-eclipse-temurin-21' 
                            args '-u root:root' 
                        }
                    }
                    steps {
                        dir('back_end/demo') {
                            echo 'Running Java tests and Verbose SonarQube analysis...'
                            
                            // Explicitly using installationName to prevent argument errors
                            withSonarQubeEnv(installationName: 'sonar-server') {
                                sh 'mvn clean test -Dmaven.test.failure.ignore=true -X' 
                                sh 'mvn sonar:sonar -Dsonar.verbose=true -Dsonar.qualitygate.wait=true' 
                            }
                        }
                    }
                }
                
                stage('Angular Frontend Testing') {
                    agent {
                        docker { 
                            image 'node:22' 
                            args '-u root:root'
                        }
                    }
                    steps {
                        dir('demo_frontend') {
                            echo 'Running Angular tests & Verbose Sonar Scan...'
                            sh 'npm install --loglevel verbose'
                            sh 'npm run test -- --watch=false' 
                            
                            withSonarQubeEnv(installationName: 'sonar-server') {
                                sh '''
                                npx sonarqube-scanner \
                                  -X \
                                  -Dsonar.projectKey=demo-frontend \
                                  -Dsonar.sources=src \
                                  -Dsonar.host.url=https://cathouse-doing-extended.ngrok-free.dev \
                                  -Dsonar.qualitygate.wait=true
                                '''
                            }
                        }
                    }
                }
                
            }
        }
    }
}
