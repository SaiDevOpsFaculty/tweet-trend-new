// Jenkins pipeline for building, testing, and deploying a Java project with Docker and Artifactory

// Define the Docker registry URL and image name with version
def registry = 'https://saidemy.jfrog.io'
def imageName = 'saidemy.jfrog.io/valaxy-docker-local/ttrend'
def version   = '2.1.2'

pipeline {
    // Define the agent to run the pipeline on a node labeled 'maven'
    agent {
        node {
            label 'maven'
        }
    }
    
    // Set environment variables for the pipeline
    environment {
        // Add Maven to the system PATH
        PATH = "/opt/apache-maven-3.9.8/bin:${env.PATH}"
    }
    
    // Define the stages of the pipeline
    stages {
        
        // Stage 1: Build the project
        stage('Build') {
            steps {
                echo "=========== Build Stage ==========="
                echo "----------- Build Started ----------"
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                echo "----------- Build Completed ----------"
            }
        }
        
        // Stage 2: Run unit tests
        stage('Test') {
            steps {
                echo "=========== Test Stage ==========="
                echo "----------- Unit Test Started ----------"
                sh 'mvn surefire-report:report'
                echo "----------- Unit Test Completed ----------"
            }
        }

        // Stage 3: Analyze code with SonarQube
        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'valaxy-sonar-scanner'
            }
            steps {
                echo "=========== SonarQube Analysis Stage ==========="
                withSonarQubeEnv('valaxy-sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        // Stage 4: Check SonarQube quality gate
        stage('Quality Gate') {
            steps {
                script {
                    echo "=========== Quality Gate Stage ==========="
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        // Stage 5: Publish the JAR files to Artifactory
        stage('Jar Publish') {
            steps {
                script {
                    echo "=========== Jar Publish Stage ==========="
                    echo '<--------------- Jar Publish Started --------------->'
                    def server = Artifactory.newServer(url: registry + "/artifactory", credentialsId: "artifact-cred")
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}"
                    def uploadSpec = """{
                          "files": [
                            {
                              "pattern": "jarstaging/(*)",
                              "target": "sai-libs-release-local/{1}",
                              "flat": "false",
                              "props": "${properties}",
                              "exclusions": [ "*.sha1", "*.md5"]
                            }
                         ]
                     }"""
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    echo '<--------------- Jar Publish Ended --------------->'   
                }
            }
        }

        // Stage 6: Build the Docker image
        stage('Docker Build') {
            steps {
                script {
                    echo "=========== Docker Build Stage ==========="
                    echo '<--------------- Docker Build Started --------------->'
                    app = docker.build(imageName + ":" + version)
                    echo '<--------------- Docker Build Ends --------------->'
                }
            }
        }

        // Stage 7: Publish the Docker image to the registry
        stage('Docker Publish') {
            steps {
                script {
                    echo "=========== Docker Publish Stage ==========="
                    echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'artifact-cred') {
                        app.push()
                    }    
                    echo '<--------------- Docker Publish Ended --------------->'  
                }
            }
        }

        // Stage 8: Deploy the application using Helm
        stage("Helm Deploy") {
            steps {
                script {
                    echo '<--------------- Helm Deploy Started --------------->'
                    sh 'helm install ttrend ttrend-0.1.0.tgz'
                    echo '<--------------- Helm Deploy Ended --------------->'
                }
            }
        }
    }
}

