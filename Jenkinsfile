pipeline {
    agent any
    stages {
        stage('Build Docker Images') {
            steps {
                sh 'docker build -t backend .'
                sh 'docker build -t frontend .'
            }
        }
        stage('Push to ECR') {
            steps {
                sh 'docker push <account_id>.dkr.ecr.<region>.amazonaws.com/backend'
                sh 'docker push <account_id>.dkr.ecr.<region>.amazonaws.com/frontend'
            }
        }
    }
}
