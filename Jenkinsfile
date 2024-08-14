// Define the URL of the Artifactory registry
def registry = 'https://saidemy.jfrog.io'

pipeline {
    // Define the agent to run the pipeline on a node with the label 'maven'
    agent {
        node {
            label 'maven'
        }
    }

    // Set environment variables for the pipeline
    environment {
        PATH = "/opt/apache-maven-3.9.8/bin:$PATH"
    }

    // Define the stages of the pipeline
    stages {
        // Stage for building the project
        stage("build") {
            steps {
                // Log message to indicate build start
                echo "----------- build started ----------"
                // Run Maven clean and deploy commands, skipping tests
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                // Log message to indicate build completion
                echo "----------- build completed ----------"
            }
        }

        // Stage for running unit tests
        stage("test") {
            steps {
                // Log message to indicate unit test start
                echo "----------- unit test started ----------"
                // Run Maven Surefire report
                sh 'mvn surefire-report:report'
                // Log message to indicate unit test completion
                echo "----------- unit test completed ----------"
            }
        }

        // Stage for SonarQube analysis
        stage('SonarQube analysis') {
            environment {
                // Set the SonarQube scanner tool
                scannerHome = tool 'valaxy-sonar-scanner'
            }
            steps {
                // Execute SonarQube analysis within the SonarQube environment
                withSonarQubeEnv('valaxy-sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        // Stage for Quality Gate check
        stage("Quality Gate") {
            steps {
                script {
                    // Set a timeout for the quality gate check
                    timeout(time: 1, unit: 'HOURS') {
                        // Wait for the quality gate result
                        def qg = waitForQualityGate()
                        // Check if the quality gate status is not OK
                        if (qg.status != 'OK') {
                            // Abort the pipeline if quality gate fails
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        // Stage for publishing the JAR files to Artifactory
        stage("Jar Publish") {
            steps {
                script {
                    // Log message to indicate JAR publish start
                    echo '<--------------- Jar Publish Started --------------->'
                    // Define the Artifactory server
                    def server = Artifactory.newServer url: registry + "/artifactory", credentialsId: "artifact-cred"
                    // Set properties for the build
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}"
                    // Define the upload specification
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
                    // Upload the files to Artifactory and collect build info
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    // Publish the build information to Artifactory
                    server.publishBuildInfo(buildInfo)
                    // Log message to indicate JAR publish end
                    echo '<--------------- Jar Publish Ended --------------->'
                }
            }
        }
    }
}

