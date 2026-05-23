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
            parallel {
                
                stage('Spring Boot (Java 21) & SonarQube') {
                    agent {
                        docker { 
                            // Using the official Java 21 Maven container
                            image 'maven:3.9-eclipse-temurin-21' 
                            args '-u root:root' 
                        }
                    }
                    steps {
                        dir('back_end/demo') {
                            echo 'Running Java tests and SonarQube analysis...'
                            
                            // This wrapper injects the URL and Token we configured in Phase 3
                            withSonarQubeEnv('sonar-server') {
                                // Runs your unit tests and triggers the Sonar scanner
                                sh 'mvn clean test sonar:sonar' 
                            }
                        }
                    }
                }
                
                stage('Angular Frontend Testing') {
                    agent {
                        docker { 
                            image 'node:18' 
                            args '-u root:root'
                        }
                    }
                    steps {
                        dir('demo_frontend') {
                            echo 'Running Angular tests...'
                            sh 'npm install'
                            // Runs the Angular test suite once without watching
                            // Note: If you get a "Chrome not found" error, your Angular karma.conf.js 
                            // needs to be configured to use ChromeHeadless for CI pipelines.
                            sh 'npm run test -- --watch=false' 
                        }
                    }
                }
                
            }
        }
    }
}
