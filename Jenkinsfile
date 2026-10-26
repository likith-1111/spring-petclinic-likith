pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        AWS_REGION        = 'us-east-1'
        AWS_ACCOUNT_ID    = '141145000253'
        ECR_REPOSITORY    = 'test-project1'
        ECR_REGISTRY      = '141145000253.dkr.ecr.us-east-1.amazonaws.com'
        ECR_IMAGE         = '141145000253.dkr.ecr.us-east-1.amazonaws.com/test-project1'

        K8S_NAMESPACE     = 'petclinic'
        HELM_RELEASE      = 'petclinic-helm'
        HELM_CHART        = './petclinic-chart'

        TRIVY_CACHE_DIR   = '/mnt/docker-data/trivy-cache'

        SONAR_PROJECT_KEY = 'spring-petclinic-likith'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out Spring PetClinic from develop'

                git branch: 'develop',
                    url: 'https://github.com/likith-1111/spring-petclinic-likith.git'

                script {
                    env.GIT_SHORT_COMMIT = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHORT_COMMIT}"
                }

                sh '''
                    set -eu

                    echo "Current branch:"
                    git branch --show-current

                    echo "Current commit:"
                    git rev-parse --short HEAD

                    echo "Docker image:"
                    echo "$ECR_IMAGE:$IMAGE_TAG"

                    echo "Disk space:"
                    df -h / /mnt/docker-data

                    test -f pom.xml
                    test -f Dockerfile
                    test -f "$HELM_CHART/Chart.yaml"
                    test -f "$HELM_CHART/templates/deployment.yaml"
                '''
            }
        }

        stage('Maven Build and Test') {
            steps {
                echo 'Building the application and running tests'

                sh '''
                    set -eu
                    mvn -B clean verify
                '''
            }

            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
                    )
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo 'Scanning project files for security findings'

                sh '''
                    set -eu

                    mkdir -p "$TRIVY_CACHE_DIR"
                    test -w "$TRIVY_CACHE_DIR"

                    trivy fs \
                        --cache-dir "$TRIVY_CACHE_DIR" \
                        --scanners vuln,misconfig,secret \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis'

                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -eu

                        mvn -B sonar:sonar \
                            -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                            -Dsonar.projectName="$SONAR_PROJECT_KEY"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for the SonarQube Quality Gate'

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Helm Chart Validation') {
            steps {
                echo 'Validating Helm chart and rendered Kubernetes manifests'

                sh '''
                    set -eu

                    helm lint "$HELM_CHART"

                    if ! grep -q 'imagePullSecrets:' \
                        "$HELM_CHART/templates/deployment.yaml"; then
                        echo "ERROR: imagePullSecrets is missing from the deployment template."
                        echo "Add imagePullSecrets under the Pod spec and commit the fix to develop."
                        exit 1
                    fi

                    if ! grep -q 'ecr-registry-secret' \
                        "$HELM_CHART/templates/deployment.yaml"; then
                        echo "ERROR: The deployment template does not reference ecr-registry-secret."
                        exit 1
                    fi

                    helm template "$HELM_RELEASE" "$HELM_CHART" \
                        --namespace "$K8S_NAMESPACE" \
                        --set-string image.repository="$ECR_IMAGE" \
                        --set-string image.tag="$IMAGE_TAG" \
                        --set-string image.pullPolicy=IfNotPresent \
                        > rendered-petclinic.yaml

                    echo "Checking rendered image:"
                    if ! grep -Fq "$ECR_IMAGE:$IMAGE_TAG" rendered-petclinic.yaml; then
                        echo "ERROR: Expected image was not found in rendered manifests."
                        grep -n 'image:' rendered-petclinic.yaml || true
                        exit 1
                    fi

                    grep -F "$ECR_IMAGE:$IMAGE_TAG" rendered-petclinic.yaml

                    echo "Checking rendered ECR pull secret:"
                    if ! grep -q 'imagePullSecrets:' rendered-petclinic.yaml; then
                        echo "ERROR: imagePullSecrets is missing from rendered manifests."
                        exit 1
                    fi

                    if ! grep -q 'name: ecr-registry-secret' rendered-petclinic.yaml; then
                        echo "ERROR: ecr-registry-secret is missing from rendered manifests."
                        grep -n -A 5 -B 5 'imagePullSecrets' rendered-petclinic.yaml || true
                        exit 1
                    fi

                    grep -n -A 4 -B 3 'imagePullSecrets' rendered-petclinic.yaml

                    echo "Helm chart validation passed."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building the PetClinic Docker image'

                sh '''
                    set -eu

                    docker build \
                        --pull \
                        -t "$ECR_IMAGE:$IMAGE_TAG" \
                        .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo 'Scanning the Docker image for security findings'

                sh '''
                    set -eu

                    trivy image \
                        --cache-dir "$TRIVY_CACHE_DIR" \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        "$ECR_IMAGE:$IMAGE_TAG"
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                echo 'Authenticating with Amazon ECR and pushing the image'

                sh '''
                    set -eu

                    aws sts get-caller-identity

                    aws ecr get-login-password \
                        --region "$AWS_REGION" \
                    | docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                    docker push "$ECR_IMAGE:$IMAGE_TAG"

                    echo "Image pushed successfully:"
                    echo "$ECR_IMAGE:$IMAGE_TAG"
                '''
            }
        }

        stage('Kubernetes Pre-deployment Checks') {
            steps {
                echo 'Checking cluster access and refreshing ECR credentials'

                sh '''
                    set -eu

                    echo "Checking Kubernetes context:"
                    kubectl config current-context

                    echo "Checking cluster nodes:"
                    kubectl get nodes

                    echo "Checking namespace:"
                    kubectl get namespace "$K8S_NAMESPACE"

                    echo "Checking application database secret:"
                    kubectl get secret petclinic-db-secret \
                        --namespace "$K8S_NAMESPACE"

                    echo "Checking Helm release:"
                    helm status "$HELM_RELEASE" \
                        --namespace "$K8S_NAMESPACE"

                    aws sts get-caller-identity

                    echo "Refreshing ECR pull secret:"
                    ECR_PASSWORD="$(aws ecr get-login-password \
                        --region "$AWS_REGION")"

                    kubectl create secret docker-registry ecr-registry-secret \
                        --namespace "$K8S_NAMESPACE" \
                        --docker-server="$ECR_REGISTRY" \
                        --docker-username=AWS \
                        --docker-password="$ECR_PASSWORD" \
                        --dry-run=client \
                        -o yaml \
                    | kubectl apply -f -

                    unset ECR_PASSWORD

                    echo "ECR pull secret refreshed successfully."
                '''
            }
        }

        stage('Helm Deploy') {
            steps {
                echo 'Deploying PetClinic using Helm'

                sh '''
                    set -eu

                    helm upgrade --install "$HELM_RELEASE" "$HELM_CHART" \
                        --namespace "$K8S_NAMESPACE" \
                        --set-string image.repository="$ECR_IMAGE" \
                        --set-string image.tag="$IMAGE_TAG" \
                        --set-string image.pullPolicy=IfNotPresent \
                        --rollback-on-failure \
                        --wait \
                        --timeout 5m
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying the PetClinic deployment'

                sh '''
                    set -eu

                    kubectl rollout status \
                        "deployment/$HELM_RELEASE" \
                        --namespace "$K8S_NAMESPACE" \
                        --timeout=300s

                    echo "Deployment:"
                    kubectl get deployment "$HELM_RELEASE" \
                        --namespace "$K8S_NAMESPACE"

                    echo "Pods:"
                    kubectl get pods \
                        --namespace "$K8S_NAMESPACE" \
                        -o wide

                    echo "Services:"
                    kubectl get services \
                        --namespace "$K8S_NAMESPACE"

                    echo "Helm release:"
                    helm status "$HELM_RELEASE" \
                        --namespace "$K8S_NAMESPACE"

                    echo "Deployment verification completed."
                '''
            }
        }
    }

    post {
        success {
            echo """
                PIPELINE SUCCESS

                Image: ${env.ECR_IMAGE}:${env.IMAGE_TAG}
                Namespace: ${env.K8S_NAMESPACE}
                Helm release: ${env.HELM_RELEASE}

                Trivy findings were reported but did not block the build.
            """
        }

        failure {
            echo """
                PIPELINE FAILED

                Inspect the first failed stage in Console Output.
                Do not delete existing Kubernetes workloads to troubleshoot.
            """
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
