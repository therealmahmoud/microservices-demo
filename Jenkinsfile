def SERVICES = [
    'frontend', 'emailservice', 'checkoutservice',
    'paymentservice', 'productcatalogservice',
    'recommendationservice', 'shippingservice',
    'adservice', 'currencyservice', 'cartservice'
]
def CHANGED_SERVICES = []

pipeline {
    agent any
    
    options {
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    environment {
        DOCKER_USER = "therealmahmoud"
        DOCKER_CLIENT_TIMEOUT = '900'
        COMPOSE_HTTP_TIMEOUT= '300'
        DOCKER_BUILDKIT = '1'
        KUBECONFIG = credentials('kube-config-id')
        K8S_NAMESPACE = "default"
    }

stages {

    stage('Detect Changed Services') {
        steps {
            script {

                
                def changedFiles = sh(
                    script: "git diff --name-only HEAD~1 HEAD",
                    returnStdout: true
                ).trim()


                echo "Changed files: ${changedFiles}"

                for (svc in SERVICES) {
                    if (changedFiles.contains("src/${svc}")) {
                        CHANGED_SERVICES.add(svc)
                    }
                }

                if (CHANGED_SERVICES.isEmpty()) {
                    echo "No service changes detected. Building all services."
                    CHANGED_SERVICES.addAll(SERVICES)
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

    stage('Build & Push (One by One)') {
            steps {
                script {
                    for (svc in CHANGED_SERVICES) {
                        echo "--- Processing: ${svc} ---"

                        sh """
                            docker build --network=host \
                            --add-host services.gradle.org:104.18.191.9 \
                             -t ${DOCKER_USER}/${svc}:latest ./src/${svc}
                        """
\
                        retry(10) { // Increased to 10 retries
                            try {
                                echo "Pushing ${svc}... Jenkins is handling it, stay relaxed."
                                sh "docker push ${DOCKER_USER}/${svc}:latest"
                            } catch (Exception e) {
                                echo "Network hiccup detected for ${svc}. Sleeping 10s and retrying automatically..."
                                sleep 10
                                error "Retrying push..." // This triggers the 'retry' block
                        }
                    }
                }
            }
        }
    }
    stage('Deploy to Kubernetes') {
        steps {
            script {
                for (svc in CHANGED_SERVICES) {
                    // Keep local scoping consistent for safety
                    def currentSvc = svc 
                    
                    echo "Updating image and applying manifests for: ${currentSvc}"
                    
                    sh """
                        # Update the image tag in the YAML file
                        sed -i 's|image:.*|image: ${DOCKER_USER}/${svc}:latest|' kubernetes-manifests/${svc}.yaml
                        
                        # Apply the manifest
                        kubectl apply -f kubernetes-manifests/${currentSvc}.yaml -n ${K8S_NAMESPACE}
                    """
                }

                echo "Waiting for rollouts to complete..."
                for (svc in CHANGED_SERVICES) {
                    def currentSvc = svc
                    sh "kubectl rollout status deployment/${currentSvc} -n ${K8S_NAMESPACE} --timeout=600s"
                }
            }
        }
    }

    stage('Apply Autoscaling') {
        steps {
            script {
                for (svc in CHANGED_SERVICES) {
                    def currentSvc = svc
                    echo "Configuring HPA for ${currentSvc}..."
                    
                    // Idempotent HPA: Try to create, if exists, patch the maxReplicas
                    sh """
                        kubectl autoscale deployment ${currentSvc} --cpu-percent=50 --min=1 --max=5 -n ${K8S_NAMESPACE} || \
                        kubectl patch hpa ${currentSvc} -n ${K8S_NAMESPACE} -p '{"spec":{"maxReplicas":5}}'
                    """
                    }
                }
            }
        }
    }
}
