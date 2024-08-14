// Jenkins Pipeline Script

def registry = 'https://saidemy.jfrog.io'
def imageName = 'saidemy.jfrog.io/valaxy-docker-local/ttrend'
def version   = '2.1.2'

pipeline {
    agent {
        node {
            label 'maven' // Specify the node label where this pipeline will run
        }
    }

    environment {
        PATH = "/opt/apache-maven-3.9.8/bin:${env.PATH}" // Set the PATH environment variable to include Maven
    }

    stages {

        // Stage 1: Build
        stage('Build') {
            steps {
                echo "----------- Build started ----------"
                sh 'mvn clean deploy -Dmaven.test.skip=true' // Run Maven build and deploy, skipping tests
                echo "----------- Build completed ----------"
            }
        }

        // Stage 2: Unit Test
        stage('Test') {
            steps {
                echo "----------- Unit test started ----------"
                sh 'mvn surefire-report:report' // Generate test reports using Maven Surefire plugin
                echo "----------- Unit test completed ----------"
            }
        }

        // Stage 3: SonarQube Analysis
        stage('SonarQube analysis') {
            environment {
                scannerHome = tool 'valaxy-sonar-scanner' // Set the SonarQube scanner home
            }
            steps {
                withSonarQubeEnv('valaxy-sonarqube-server') { 
                    // Analyze the project with SonarQube
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        // Stage 4: Quality Gate Check
        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') { 
                        // Set a timeout for the quality gate check
                        def qg = waitForQualityGate() // Wait for SonarQube quality gate result
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}" // Fail pipeline if quality gate fails
                        }
                    }
                }
            }
        }

        // Stage 5: Jar Publish
        stage('Jar Publish') {
            steps {
                script {
                    echo '<--------------- Jar Publish Started --------------->'
                    
                    // Define the Artifactory server and credentials
                    def server = Artifactory.newServer(url: registry + "/artifactory", credentialsId: "artifact-cred")
                    
                    // Define properties and upload specification for the JAR files
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}"
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "jarstaging/(*)",
                                "target": "libs-release-local/{1}",
                                "flat": "false",
                                "props": "${properties}",
                                "exclusions": [ "*.sha1", "*.md5"]
                            }
                        ]
                    }"""
                    
                    // Upload the JAR files to Artifactory
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect() // Collect environment variables for build info
                    server.publishBuildInfo(buildInfo) // Publish the build info to Artifactory
                    
                    echo '<--------------- Jar Publish Ended --------------->'
                }
            }
        }

        // Stage 6: Docker Build
        stage('Docker Build') {
            steps {
                script {
                    echo '<--------------- Docker Build Started --------------->'
                    
                    // Build Docker image with the specified name and version
                    app = docker.build(imageName + ":" + version)
                    
                    echo '<--------------- Docker Build Ends --------------->'
                }
            }
        }

        // Stage 7: Docker Publish
        stage('Docker Publish') {
            steps {
                script {
                    echo '<--------------- Docker Publish Started --------------->'
                    
                    // Publish Docker image to the specified registry
                    docker.withRegistry(registry, 'artifact-cred') {
                        app.push()
                    }
                    
                    echo '<--------------- Docker Publish Ended --------------->'
                }
            }
        }

        // Stage 8: Deploy
        stage('Deploy') {
            steps {
                script {
                    // Execute the deployment script
                    sh './deploy.sh'
                }
            }
        }
    }
}

