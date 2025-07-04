# vprofile-CI-pipeline

- integrating Nexus to send and store repo, got pipeline from Nexus Artifact uploader on website
- integrated slack for sending of notifications if build failed or not

def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

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
        stage('Unit Test'){
            steps{
                sh 'mvn test'
            }
        }
        stage('Checkstyle Analysis'){
            steps{
                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage("Sonar Code Analysis") {
        	environment {
                scannerHome = tool 'Sonar6.2'
            }
            steps {
              withSonarQubeEnv('sonarserver') {
                sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml '''
              }
            }
        }
        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        stage("uploadArtifacts"){
            steps{
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.83.110:8081',
                    groupId: 'quality assurance',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-repo',
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [artifactId: 'The_vproApp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war']
                        ]
                    )
            }
            
        }

	}
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#devops_cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }

}
