pipeline {
    agent any 
    
    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling monorepo code from GitHub...'
                checkout scm
            }
        }
        
        stage('Concurrent Build, Analyze & Deploy') {
            parallel {
                
                // ==========================================
                // TRACK 1: SPRING BOOT (Java 21)
                // ==========================================
                stage('Spring Boot Backend') {
                    agent {
                        docker { 
                            image 'maven:3.9-eclipse-temurin-21' 
                            args '-u root:root -v maven-cache:/root/.m2 -v sonar-cache:/root/.sonar/cache' 
                        }
                    }
                    steps {
                        dir('back_end/demo') {
                            echo 'Compiling Java, running tests, and pushing to SonarQube...'
                            
                            withSonarQubeEnv(installationName: 'sonar-server') {
                                sh 'mvn clean package -Dmaven.test.failure.ignore=true -X' 
                                sh 'mvn sonar:sonar -Dsonar.verbose=true -Dsonar.qualitygate.wait=true' 
                            }
                            
                            echo 'Backend SonarQube passed. Uploading JAR to Nexus...'
                            
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                // **IMPORTANT: Ensure this is your actual Nexus VM IP**
                                nexusUrl: '192.168.56.31:8081', 
                                groupId: 'com.demo',
                                version: '1.0.0',
                                repository: 'maven-releases', 
                                credentialsId: 'nexus-credentials',
                                artifacts: [
                                    [
                                        artifactId: 'backend-demo', 
                                        classifier: '', 
                                        file: 'target/demo-0.0.1-SNAPSHOT.jar', 
                                        type: 'jar'
                                    ]
                                ]
                            )
                        }
                    }
                }
                
                // ==========================================
                // TRACK 2: ANGULAR FRONTEND (Node 22)
                // ==========================================
                stage('Angular Frontend') {
                    agent {
                        docker { 
                            image 'node:22' 
                            args '-u root:root -v npm-cache:/root/.npm -v sonar-cache:/root/.sonar/cache'
                        }
                    }
                    steps {
                        dir('demo_frontend') {
                            echo 'Building Angular, running tests, and pushing to SonarQube...'
                            
                            sh 'npm install --loglevel verbose'
                            sh 'npm run build'
                            sh 'npm run test -- --watch=false' 
                            
                            withSonarQubeEnv(installationName: 'sonar-server') {
                                sh '''
                                npx sonarqube-scanner \
                                  -X \
                                  -Dsonar.projectKey=demo-frontend \
                                  -Dsonar.sources=src \
                                  -Dsonar.host.url=https://cathouse-doing-extended.ngrok-free.dev \
                                  -Dsonar.qualitygate.wait=true \
                                  -Dsonar.javascript.node.maxspace=4096
                                '''
                            }
                            
                            echo 'Frontend SonarQube passed. Compressing and uploading to Nexus...'
                            
                            sh 'tar -czvf frontend-app.tar.gz dist/'
                            
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                // **IMPORTANT: Ensure this is your actual Nexus VM IP**
                                nexusUrl: '192.168.56.31:8081', 
                                groupId: 'com.demo',
                                version: '1.0.0',
                                repository: 'frontend-releases', 
                                credentialsId: 'nexus-credentials',
                                artifacts: [
                                    [
                                        artifactId: 'angular-frontend', 
                                        classifier: '', 
                                        file: 'frontend-app.tar.gz', 
                                        type: 'tar.gz'
                                    ]
                                ]
                            )
                        }
                    }
                }
                
            }
        }
    }
}
