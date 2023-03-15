pipeline{

    agent any

    stages{
        stage('pull code'){
            steps{
                git credentialsId: '28e2f244-7819-4500-a12a-1ab0a1855aea', url: 'https://github.com/DennnnyX/pipeline.git'
                echo 'pull code'
            }
        }
        stage('compile'){
            steps{
                sh "mvn clean package"
                echo 'compile'
            }
        }
        stage('run'){
            steps{
                echo 'run'
            }
        }


    }
}