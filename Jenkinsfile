def SERVICES = [
    'frontend', 'cartservice', 'checkoutservice',
    'paymentservice', 'productcatalogservice',
    'recommendationservice', 'shippingservice',
    'adservice', 'currencyservice', 'emailservice'
]
def CHANGED_SERVICES = []

pipeline {
    agent any
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    environment {
        DOCKER_USER = "therealmahmoud"
        DOCKER_CLIENT_TIMEOUT = '900'
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

    stage('Build & Push Images (Limited Parallel)') {
        steps {
            script {
                // Set limit to 2 services at a time
                def batchSize = 2 
                
                for (int i = 0; i < CHANGED_SERVICES.size(); i += batchSize) {
                    
                    // FIX: Get the subList, then immediately cast it into a standard ArrayList
                    // so Jenkins can serialize it without throwing an exception.
                    def rawSubList = CHANGED_SERVICES.subList(i, Math.min(i + batchSize, CHANGED_SERVICES.size()))
                    def batch = new ArrayList(rawSubList) 
                    
                    def builds = [:]
                    
                    for (svc in batch) {
                        def currentSvc = svc 
                        builds[currentSvc] = {
                            echo "Building and pushing ${currentSvc}"
                            sh """
                                docker build -t ${DOCKER_USER}/${currentSvc}:latest ./src/${currentSvc}
                                docker push ${DOCKER_USER}/${currentSvc}:latest
                            """
                        }
                    }
                    echo "Running batch: ${batch}"
                    parallel builds
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
                        sed -i "s|image:.*${currentSvc}:.*|image: ${DOCKER_USER}/${currentSvc}:latest|" kubernetes-manifests/${currentSvc}.yaml
                        
                        # Apply the manifest
                        kubectl apply -f kubernetes-manifests/${currentSvc}.yaml -n ${K8S_NAMESPACE}
                    """
                }

                echo "Waiting for rollouts to complete..."
                for (svc in CHANGED_SERVICES) {
                    def currentSvc = svc
                    sh "kubectl rollout status deployment/${currentSvc} -n ${K8S_NAMESPACE} --timeout=120s"
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
