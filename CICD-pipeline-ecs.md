# cicd-pipeline-ecs

- The Docker engine runs ec2 istance or vm, which doesn't offer production related features like high-availability, self-healing, etc...
- deploy container on ecs, which is more scalable, secure and has more features then ecr
- had to install "aws steps" plugin to upload to ecs on jenkins

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
    environment {
        registryCredential = 'ecr:us-east-1:awscred'
        appRegistry = "657349740669.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg"
        vprofileRegistry = "https://657349740669.dkr.ecr.us-east-1.amazonaws.com"
        cluster = 'the_vprofile'
        service = 'vproappSVC'
    }

    stages{
        stage('Fetch Code'){
            steps{
                git branch: 'docker', url: 'https://github.com/hkhcoder/vprofile-project.git'
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
        stage('Build App Image') {
          steps {
       
            script {
                dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
          }
    
        }

        stage('Upload App Image') {
          steps{
            script {
              docker.withRegistry( vprofileRegistry, registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }
        stage('remove container from jenkins'){
            steps{
                sh 'docker rmi -f $(docker images -q)'
            }
        }
        stage('deploy to ecs'){
            steps{
                withAWS(credentials: 'awscred', region: 'us-east-1'){
                    sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                }
            }
        }
    }
    
}
