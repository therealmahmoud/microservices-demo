pipeline {
    agent any
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    environment {
        DOCKER_USER = "therealmahmoud"
        IMAGE_NAME = "frontend"
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
                            script {
                                def imageExists = sh(
                                    script: "docker pull ${DOCKER_USER}/${svc}:latest || true",
                                    returnStatus: true
                                ) == 0

                                if (!imageExists) {
                                    echo "Image ${DOCKER_USER}/${svc}:latest not found. Building..."
                                    sh """
                                        docker build -t ${DOCKER_USER}/${svc}:latest ./src/${svc}
                                        docker push ${DOCKER_USER}/${svc}:latest
                                    """
                                } else {
                                    echo "Image ${DOCKER_USER}/${svc}:latest already exists on Docker Hub."
                                }
                            }
                        }
                    }
                    parallel builds
                }
            }
        }

    stage('Deploy to Kubernetes') {
    steps {
        script {

            echo "Updating image tags..."

            for (svc in CHANGED_SERVICES) {

                sh """
                sed -i "s|image:.*${svc}:.*|image: ${DOCKER_USER}/${svc}:latest|"\
                kubernetes-manifests/${svc}.yaml
                """
            }

            echo "Applying Kubernetes manifests..."

            sh """
            kubectl apply -k kubernetes-manifests/ -n ${K8S_NAMESPACE}
            """

            echo "Waiting for rollouts..."

            for (svc in CHANGED_SERVICES) {
                sh """
                kubectl rollout status deployment/${svc} -n ${K8S_NAMESPACE} --timeout=120s
                """
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
