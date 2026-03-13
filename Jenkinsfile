pipeline {
    agent any
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    environment {
        DOCKER_USER = "therealmahmoud"
        IMAGE_NAME = "frontend"
        IMAGE_TAG = "${GIT_COMMIT}"
        DOCKER_CLIENT_TIMEOUT = '600'
        COMPOSE_HTTP_TIMEOUT = '600'
        DOCKER_BUILDKIT = '1'
    }

stages {

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
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

                        sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        '''

                        for (svc in services) {

                            sh """
                            echo "Building $svc..."

                            docker pull $DOCKER_USER/$svc:latest || true
                            docker build \
                                --cache-from=$DOCKER_USER/$svc:latest \
                                -t $DOCKER_USER/$svc:$IMAGE_TAG \
                                -t $DOCKER_USER/$svc:latest \
                                ./src/$svc
                            echo "Pushing $svc..."
                            
                            # Retry logic for pushing images to handle transient network issues with Docker Hub

                            for i in 1 2 3; do
                                docker push $DOCKER_USER/$svc:$IMAGE_TAG && break
                                echo "Retry push..."
                                sleep 20
                            done

                            for i in 1 2 3; do
                                docker push $DOCKER_USER/$svc:latest && break
                                echo "Retry push..."
                                sleep 20
                            done
                            """
                        }

                        sh "docker logout"
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
