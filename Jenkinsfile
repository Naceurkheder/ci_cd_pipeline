pipeline {
    agent any

    environment {
        NEXUS_URL          = 'http://192.168.56.31:8081'
        NEXUS_NPM_REGISTRY = 'http://192.168.56.31:8081/repository/npm-proxy/'
        NEXUS_MAVEN_REPO   = 'http://192.168.56.31:8081/repository/maven-public/'
        SONAR_HOST_URL     = 'https://cathouse-doing-extended.ngrok-free.dev'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Pulling monorepo code from GitHub...'
                checkout scm
            }
        }

        stage('Concurrent Build, Analyze & Deploy') {
            parallel {
                stage('Spring Boot Backend') {
                    agent {
                        docker {
                            image 'maven:3.9-eclipse-temurin-21'
                            reuseNode true
                            args '-v sonar-cache:/root/.sonar/cache -v maven-cache:/root/.m2 -u root:root'
                        }
                    }
                    steps {
                        dir('back_end/demo') {
                            withCredentials([usernamePassword(
                                credentialsId: 'nexus-credentials',
                                usernameVariable: 'NEXUS_USER',
                                passwordVariable: 'NEXUS_PASS'
                            )]) {
                                writeFile file: 'settings.xml', text: """<?xml version="1.0" encoding="UTF-8"?>
                                        <settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
                                                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                                                  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0
                                                                      https://maven.apache.org/xsd/settings-1.2.0.xsd">
                                        
                                          <servers>
                                            <server>
                                              <id>nexus</id>
                                              <username>${NEXUS_USER}</username>
                                              <password>${NEXUS_PASS}</password>
                                            </server>
                                          </servers>
                                        
                                          <mirrors>
                                            <mirror>
                                              <id>nexus</id>
                                              <mirrorOf>*</mirrorOf>
                                              <url>${NEXUS_MAVEN_REPO}</url>
                                            </mirror>
                                          </mirrors>
                                        
                                          <profiles>
                                            <profile>
                                              <id>nexus</id>
                                              <repositories>
                                                <repository>
                                                  <id>central</id>
                                                  <url>${NEXUS_MAVEN_REPO}</url>
                                                  <releases><enabled>true</enabled></releases>
                                                  <snapshots><enabled>true</enabled></snapshots>
                                                </repository>
                                              </repositories>
                                              <pluginRepositories>
                                                <pluginRepository>
                                                  <id>central</id>
                                                  <url>${NEXUS_MAVEN_REPO}</url>
                                                  <releases><enabled>true</enabled></releases>
                                                  <snapshots><enabled>true</enabled></snapshots>
                                                </pluginRepository>
                                              </pluginRepositories>
                                            </profile>
                                          </profiles>
                                        
                                          <activeProfiles>
                                            <activeProfile>nexus</activeProfile>
                                          </activeProfiles>
                                        
                                        </settings>"""

                                withSonarQubeEnv('sonar-server') {
                                    sh """
                                        mvn clean package \
                                            -s settings.xml \
                                            -Dmaven.test.failure.ignore=true
                                    """
                                    sh """
                                        mvn sonar:sonar \
                                            -s settings.xml \
                                            -Dsonar.projectKey=demo-backend \
                                            -Dsonar.sources=src/main/java \
                                            -Dsonar.java.binaries=target/classes \
                                            -Dsonar.qualitygate.wait=true
                                    """
                                }
                            }
                        }
                    }
                }

                stage('Angular Frontend') {
                    agent {
                        docker {
                            image 'node:22'
                            reuseNode true
                            args '-v sonar-cache:/root/.sonar/cache -v npm-cache:/root/.npm -u root:root'
                        }
                    }
                    steps {
                        dir('demo_frontend') {
                            withCredentials([usernamePassword(
                                credentialsId: 'nexus-credentials',
                                usernameVariable: 'NEXUS_USER',
                                passwordVariable: 'NEXUS_PASS'
                            )]) {
                                echo 'Setting npm registry and credentials for Nexus...'
                                sh """
                                    npm config set registry ${NEXUS_NPM_REGISTRY}
                                    npm config set //192.168.56.31:8081/repository/npm-proxy/:username ${NEXUS_USER}
                                    npm config set //192.168.56.31:8081/repository/npm-proxy/:_password \$(echo -n '${NEXUS_PASS}' | base64)
                                    npm config set //192.168.56.31:8081/repository/npm-proxy/:email ci@ci.local
                                """

                                echo 'Installing dependencies from Nexus...'
                                sh 'npm install --prefer-offline --no-audit --no-fund --loglevel warn'

                                echo 'Building Angular app...'
                                sh 'npm run build'

                                echo 'Running SonarQube scan...'
                                withSonarQubeEnv('sonar-server') {
                                    sh """
                                        npx --yes \
                                            --registry=${NEXUS_NPM_REGISTRY} \
                                            sonarqube-scanner \
                                            -Dsonar.projectKey=demo-frontend \
                                            -Dsonar.sources=src \
                                            -Dsonar.host.url=${SONAR_HOST_URL} \
                                            -Dsonar.qualitygate.wait=true \
                                            -Dsonar.javascript.node.maxspace=4096
                                    """
                                }
                            }
                        }
                    }
                }

            } // end parallel
        }

    } // end stages

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed — check stage logs above.'
        }
    }
}
