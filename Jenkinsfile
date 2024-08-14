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
    }
}

