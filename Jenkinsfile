def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any
    tools {
        jdk "oracleJdk17"
        maven "MAVEN"
    }
     environment {
        registryCredential = 'ecr:eu-north-1:awscreds'
        appRegistry = "047719647048.dkr.ecr.eu-north-1.amazonaws.com/vprofile"
        vprofileRegistry = "https://047719647048.dkr.ecr.eu-north-1.amazonaws.com"
        cluster = "vprofile1"
        service = "vprofile-service"
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'Sonar6'
            }
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=KK55 \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                success {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} completed successfully!",
                        color: '#00FF00'
                    )
                }
                failure {
                    slackSend(
                        channel: '#jenkins-cicd',
                        message: "${env.STAGE_NAME} failed!",
                        color: '#FF0000'
                    )
                }
            }
        }

        // stage('Upload Artifacts to Nexus Repo') {
        //     steps {
        //         nexusArtifactUploader(
        //             nexusVersion: 'nexus3',
        //             protocol: 'http',
        //             nexusUrl: '172.31.30.146:8081',
        //             groupId: 'QA',
        //             version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
        //             repository: 'vprofile-repo',
        //             credentialsId: 'sonarLogin',
        //             artifacts: [
        //                 [artifactId: 'vproapp',
        //                  classifier: '',
        //                  file: 'target/vprofile-v2.war',
        //                  type: 'war']
        //             ]
        //         )
        //     }
        //     post {
        //         success {
        //             slackSend(
        //                 channel: '#jenkins-cicd',
        //                 message: "${env.STAGE_NAME} completed successfully!",
        //                 color: '#00FF00'
        //             )
        //         }
        //         failure {
        //             slackSend(
        //                 channel: '#jenkins-cicd',
        //                 message: "${env.STAGE_NAME} failed!",
        //                 color: '#FF0000'
        //             )
        //         }
        //     }
        // }
            stage('Build App Image') {
             steps {
       
             script {
                dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./")
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
          stage('Remove Docker Images') {
            steps {
                script {
                    // Remove the image from the local Docker environment to save space
                    sh "docker rmi ${appRegistry}:$BUILD_NUMBER || true"
                    sh "docker rmi ${appRegistry}:latest || true"
                }
            }
        }
        stage('Deploy to ecs') {
          steps {
            withAWS(credentials: 'awscreds', region: 'eu-north-1') {
            sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                }
            }
        }

    }

    post {
        always {
            echo 'Final Slack Notification.'
            slackSend channel: '#jenkins-cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
