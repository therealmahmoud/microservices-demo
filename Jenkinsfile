pipeline {
    agent any

    environment {
        DOCKER_USER = "therealmahmoud"
        IMAGE_NAME = "frontend"
        IMAGE_TAG = "${GIT_COMMIT}"
    }

    stages {

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        def services = [
                            'frontend',
                            'cartservice',
                            'checkoutservice',
                            'paymentservice',
                            'productcatalogservice',
                            'recommendationservice',
                            'shippingservice',
                            'adservice',
                            'currencyservice',
                            'emailservice'
                        ]

                        for (svc in services) {
                            sh """
                            docker build -t $DOCKER_USER/$svc:$IMAGE_TAG ./src/$svc
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker push $DOCKER_USER/$svc:$IMAGE_TAG
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def services = [
                        'frontend',
                        'cartservice',
                        'checkoutservice',
                        'paymentservice',
                        'productcatalogservice',
                        'recommendationservice',
                        'shippingservice',
                        'adservice',
                        'currencyservice',
                        'emailservice'
                    ]

                    for (svc in services) {
                        sh """
                        kubectl set image deployment/${svc} ${svc}=$DOCKER_USER/${svc}:$IMAGE_TAG
                        kubectl rollout status deployment/${svc}
                        """
                    }
                }
            }
        }

        stage('Apply Autoscaling') {
            steps {
                script {
                    def services = [
                        'frontend',
                        'cartservice',
                        'checkoutservice',
                        'paymentservice',
                        'productcatalogservice',
                        'recommendationservice',
                        'shippingservice',
                        'adservice',
                        'currencyservice',
                        'emailservice'
                    ]

                    for (svc in services) {
                        sh """
                        kubectl autoscale deployment ${svc} --cpu-percent=50 --min=1 --max=5 || \
                        kubectl patch hpa ${svc} -p '{"spec":{"maxReplicas":5}}'
                        """
                    }
                }
            }
        }

    }
}
