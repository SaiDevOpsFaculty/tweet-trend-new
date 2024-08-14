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
                // Beginning of the build stage
                echo "=========== Build Stage ==========="
                echo "----------- Build Started ----------"
                // Execute Maven clean and deploy commands, skipping tests
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                // End of the build stage
                echo "----------- Build Completed ----------"
            }
        }
        
        // Stage 2: Run unit tests
        stage('Test') {
            steps {
                // Beginning of the test stage
                echo "=========== Test Stage ==========="
                echo "----------- Unit Test Started ----------"
                // Generate test reports using Maven Surefire plugin
                sh 'mvn surefire-report:report'
                // End of the test stage
                echo "----------- Unit Test Completed ----------"
            }
        }

        // Stage 3: Analyze code with SonarQube
        stage('SonarQube Analysis') {
            environment {
                // Set the SonarQube scanner home environment variable
                scannerHome = tool 'valaxy-sonar-scanner'
            }
            steps {
                // Beginning of the SonarQube analysis stage
                echo "=========== SonarQube Analysis Stage ==========="
                // Start SonarQube analysis
                withSonarQubeEnv('valaxy-sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        // Stage 4: Check SonarQube quality gate
        stage('Quality Gate') {
            steps {
                script {
                    // Beginning of the quality gate stage
                    echo "=========== Quality Gate Stage ==========="
                    // Start checking SonarQube quality gate
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        // Abort the pipeline if the quality gate fails
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
                    // Beginning of the JAR publishing stage
                    echo "=========== Jar Publish Stage ==========="
                    // Start of the JAR publishing process
                    echo '<--------------- Jar Publish Started --------------->'
                    // Connect to Artifactory server
                    def server = Artifactory.newServer(url: registry + "/artifactory", credentialsId: "artifact-cred")
                    // Set properties for the JAR files
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}"
                    // Define the upload specification for JAR files
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
                    // Upload the JAR files to Artifactory
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    // End of the JAR publishing process
                    echo '<--------------- Jar Publish Ended --------------->'   
                }
            }
        }

        // Stage 6: Build the Docker image
        stage('Docker Build') {
            steps {
                script {
                    // Beginning of the Docker build stage
                    echo "=========== Docker Build Stage ==========="
                    // Start Docker image build process
                    echo '<--------------- Docker Build Started --------------->'
                    app = docker.build(imageName + ":" + version)
                    // End Docker image build process
                    echo '<--------------- Docker Build Ends --------------->'
                }
            }
        }

        // Stage 7: Publish the Docker image to the registry
        stage('Docker Publish') {
            steps {
                script {
                    // Beginning of the Docker publish stage
                    echo "=========== Docker Publish Stage ==========="
                    // Start Docker image publishing process
                    echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'artifact-cred') {
                        app.push()
                    }    
                    // End Docker image publishing process
                    echo '<--------------- Docker Publish Ended --------------->'  
                }
            }
        }

        // Stage 8: Deploy the application
        stage('Deploy') {
            steps {
                script {
                    // Beginning of the deployment stage
                    echo "=========== Deploy Stage ==========="
                    // Execute the deployment script
                    sh './deploy.sh'
                    // End of the deployment stage
                    echo "----------- Deployment Completed ----------"
                }
            }
        }
    }
}

