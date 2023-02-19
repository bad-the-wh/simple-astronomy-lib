pipeline {
    agent any

    tools {
        maven 'MVN'
    }

    stages {
        stage('Build MAVEN') {
            steps{
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs:[[credentialsId: 'BAD', url: 'https://github.com/bad-the-wh/simple-astronomy-lib']]])


            }
        }
        stage('Build') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
        stage('Test') {
            steps {
                sh 'make check || true'
                junit '**/target/*.xml'
            }
        }
        stage('Deploy') {
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
                steps {
                    sh 'make publish'
                }
        }
    }
}