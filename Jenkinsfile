stage('Test & Analyze') {
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
                            withSonarQubeEnv('sonar-server') {
                                // -X adds verbose logging to Maven itself
                                sh 'mvn clean test -Dmaven.test.failure.ignore=true -X' 
                                
                                // Added verbose flag AND the Quality Gate wait flag
                                sh 'mvn sonar:sonar -Dsonar.verbose=true -Dsonar.qualitygate.wait=true' 
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
                            echo 'Running Angular tests & Verbose Sonar Scan...'
                            // Added --loglevel verbose for npm
                            sh 'npm install --loglevel verbose'
                            sh 'npm run test -- --watch=false' 
                            
                            withSonarQubeEnv('sonar-server') {
                                // Added -X for verbose scanner output and the Quality Gate wait flag
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
