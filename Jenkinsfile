pipeline {
    agent any
    
    options {
        timeout(time: 10, unit: 'MINUTES')
        timestamps()
    }

    environment {
        DOCKER_USER = "therealmahmoud"
        IMAGE_NAME = "frontend"
        IMAGE_TAG = "${GIT_COMMIT}"
    }

    stages {

        stage('Build & Push Docker Images') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
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

                        sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        """
                        
                        for (svc in services) {
                            sh """
                            set -e
                            echo "Building and pushing $svc..."
                            docker build -t $DOCKER_USER/$svc:$IMAGE_TAG ./src/$svc
                            
                            for attempt in 1 2 3; do
                                echo "Push attempt \$attempt for $svc..."
                                if docker push $DOCKER_USER/$svc:$IMAGE_TAG; then
                                    echo "Successfully pushed $svc"
                                    break
                                elif [ \$attempt -lt 3 ]; then
                                    echo "Push failed, waiting 30s before retry..."
                                    sleep 30
                                else
                                    echo "Push failed after 3 attempts for $svc"
                                    exit 1
                                fi
                            done
                            """
                        }
                        
                        sh """
                        # Logout after all pushes complete
                        docker logout
                        """
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
