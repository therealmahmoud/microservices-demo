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
        KUBECONFIG = credentials('kube-config-id')
        K8S_NAMESPACE = "default"
    }

stages {

    stage('Detect Changed Services') {
        steps {
            script {

                SERVICES = [
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

                def changedFiles = sh(
                    script: "git diff --name-only HEAD~1 HEAD",
                    returnStdout: true
                ).trim()

                echo "Changed files: ${changedFiles}"

                CHANGED_SERVICES = []

                for (svc in SERVICES) {
                    if (changedFiles.contains("src/${svc}")) {
                        CHANGED_SERVICES.add(svc)
                    }
                }

                if (CHANGED_SERVICES.isEmpty()) {
                    echo "No service changes detected. Building all services."
                    CHANGED_SERVICES = SERVICES
                }

                echo "Services to build: ${CHANGED_SERVICES}"
            }
        }
    }

    stage('Docker Login') {
        steps {
            withCredentials([usernamePassword(
                credentialsId: 'dockerhub-creds',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )]) {

                sh '''
                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                '''
            }
        }
    }

    stage('Build & Push Images (Parallel)') {
            steps {
                script {

                    def builds = [:]

                    for (svc in CHANGED_SERVICES) {

                        builds[svc] = {

                            sh """
                            set -e
                            echo "Building ${svc}"

                            docker build \
                                -t ${DOCKER_USER}/${svc}:${IMAGE_TAG} \
                                -t ${DOCKER_USER}/${svc}:latest \
                                ./src/${svc}

                            echo "Pushing ${svc}"

                            for i in 1 2 3; do
                                docker push ${DOCKER_USER}/${svc}:${IMAGE_TAG} && break
                                echo "Retry push..."
                                sleep 20
                            done

                            for i in 1 2 3; do
                                docker push ${DOCKER_USER}/${svc}:latest && break
                                echo "Retry push..."
                                sleep 20
                            done
                            """
                        }
                    }

                    parallel builds
                }
            }
        }

    stage('Deploy to Kubernetes') {
            steps {
                script {
                    for (svc in CHANGED_SERVICES) {
                        echo "Deploying ${svc}..."

                        def exists = sh(
                            script: "kubectl get deployment ${svc} -n $K8S_NAMESPACE --ignore-not-found",
                            returnStdout: true
                        ).trim()

                        if (!exists) {
                            echo "Deployment ${svc} not found. Creating..."
                            sh "kubectl create deployment ${svc} --image=${DOCKER_USER}/${svc}:${IMAGE_TAG} -n $K8S_NAMESPACE"
                        } else {
                            echo "Updating ${svc} image..."

                            def updateResult = sh(
                                script: """
                                kubectl set image deployment/${svc} ${svc}=${DOCKER_USER}/${svc}:${IMAGE_TAG} -n $K8S_NAMESPACE || \\
                                kubectl set image deployment/${svc} ${svc}=${DOCKER_USER}/${svc}:latest -n $K8S_NAMESPACE
                                """,
                                returnStatus: true
                            )

                            if (updateResult != 0) {
                                error "Failed to update ${svc} deployment with both commit tag and latest image."
                            }
                        }
                        sh "kubectl rollout status deployment/${svc} -n $K8S_NAMESPACE --timeout=120s"
                    }
                }
            }
        }

    stage('Apply Autoscaling') {
            steps {
                script {
                    for (svc in CHANGED_SERVICES) {
                        sh """
                        kubectl autoscale deployment ${svc} --cpu-percent=50 --min=1 --max=5 -n $K8S_NAMESPACE || \
                        kubectl patch hpa ${svc} -n $K8S_NAMESPACE -p '{"spec":{"maxReplicas":5}}'
                        """
                    }
                }
            }
        }
    }
}
