pipeline {
    agent any

    environment {
        DOCKER_USER = "therealmahmoud"
        IMAGE_NAME = "frontend"
        IMAGE_TAG = "${GIT_COMMIT}"
    }

    stages {

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG ./src/frontend'
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                sh '''
                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl set image deployment/frontend\
                server=$DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }
        
        stage('Apply Autoscaling') {
            steps {
                sh """
                # Create HPA if not exists, else patch max replicas
                kubectl autoscale deployment frontend --cpu-percent=50 --min=1 --max=5 || \
                kubectl patch hpa frontend -p '{"spec":{"maxReplicas":5}}'
                """
            }
        }

    }
}
