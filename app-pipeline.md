# vprofile-pipeline
- automated pipeline to fetch code from GitHub, unit test with maven, then build the artifact to store in Jenkins workspace
  


pipeline{
    agent any
    tools{
        maven 'Maven3.9'
        jdk 'JDK17'     
    }
    stages{
        stage('Fetch Code'){
            steps{
                git branch: 'atom', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }
        stage('Unit Test'){
            steps{
                sh 'mvn test'
            }
        }
        stage('Build'){
            steps{
                sh 'mvn install -DskipTests'
            }
            post{
                success{
                    echo "Archive the build artifacts"
                    archiveArtifacts artifacts: '**/*.war'                   
                }
            }
        }
    }
}
