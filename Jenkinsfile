
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
                // Run Maven clean and deploy commands
                sh 'mvn clean deploy'
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
    }
}

