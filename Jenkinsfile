pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '454143665149'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_TAG = "${BUILD_NUMBER}"

        TRIVY_CACHE_DIR = "${WORKSPACE}/.trivy-cache"
        TMPDIR           = "${WORKSPACE}/.trivy-tmp"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }

                sh '''
                    echo "Image tag: ${IMAGE_TAG}"
                '''
            }
        }

        stage('Unit Tests') {
            parallel {

                stage('UI Tests') {
                    steps {
                        dir('src/ui') {
                            sh '''
                                chmod +x gradlew
                                ./gradlew clean test
                            '''
                        }
                    }
                }

                stage('Catalog Tests') {
                    steps {
                        dir('src/catalog') {
                            sh '''
                                go test ./...
                            '''
                        }
                    }
                }

                stage('Cart Tests') {
                    steps {
                        dir('src/cart') {
                            sh '''
                                chmod +x gradlew
                                ./gradlew clean test
                            '''
                        }
                    }
                }

                stage('Orders Tests') {
                    steps {
                        dir('src/orders') {
                            sh '''
                                chmod +x gradlew
                                ./gradlew clean test
                            '''
                        }
                    }
                }

                stage('Checkout Tests') {
                    steps {
                        dir('src/checkout') {
                            sh '''
                                corepack enable
                                yarn install --immutable
                                yarn test || true
                            '''
                        }
                    }
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=retail-store-sample-app \
                          -Dsonar.projectName="Retail Store Sample App" \
                          -Dsonar.sources=src
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_REGISTRY}/retail-store-ui:${IMAGE_TAG} \
                      src/ui

                    docker build \
                      -t ${ECR_REGISTRY}/retail-store-catalog:${IMAGE_TAG} \
                      src/catalog

                    docker build \
                      -t ${ECR_REGISTRY}/retail-store-cart:${IMAGE_TAG} \
                      src/cart

                    docker build \
                      -t ${ECR_REGISTRY}/retail-store-orders:${IMAGE_TAG} \
                      src/orders

                    docker build \
                      -t ${ECR_REGISTRY}/retail-store-checkout:${IMAGE_TAG} \
                      src/checkout
                '''
            }
        }

        stage('Prepare Trivy') {
            steps {
                sh '''
                    mkdir -p "${TRIVY_CACHE_DIR}"
                    mkdir -p "${TMPDIR}"

                    trivy image \
                      --cache-dir "${TRIVY_CACHE_DIR}" \
                      --download-db-only

                    trivy image \
                      --cache-dir "${TRIVY_CACHE_DIR}" \
                      --download-java-db-only
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    for image in \
                      retail-store-ui \
                      retail-store-catalog \
                      retail-store-cart \
                      retail-store-orders \
                      retail-store-checkout
                    do
                      echo "Scanning ${image}:${IMAGE_TAG}"

                      TMPDIR="${TMPDIR}" \
                      trivy image \
                        --cache-dir "${TRIVY_CACHE_DIR}" \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --exit-code 0 \
                        --skip-version-check \
                        "${ECR_REGISTRY}/${image}:${IMAGE_TAG}"
                    done
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region "${AWS_REGION}" \
                    | docker login \
                      --username AWS \
                      --password-stdin "${ECR_REGISTRY}"
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    for image in \
                      retail-store-ui \
                      retail-store-catalog \
                      retail-store-cart \
                      retail-store-orders \
                      retail-store-checkout
                    do
                      echo "Pushing ${image}:${IMAGE_TAG}"

                      docker push \
                        "${ECR_REGISTRY}/${image}:${IMAGE_TAG}"
                    done
                '''
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-credentials',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -rf retail-store-gitops

                        git clone \
                          https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/YOUR_USERNAME/retail-store-gitops.git

                        cd retail-store-gitops

                        git config user.name "jenkins"
                        git config user.email "jenkins@example.com"

                        sed -i \
                          "s|retail-store-ui:.*|retail-store-ui:${IMAGE_TAG}|g" \
                          environments/dev/deploy.yaml

                        sed -i \
                          "s|retail-store-catalog:.*|retail-store-catalog:${IMAGE_TAG}|g" \
                          environments/dev/deploy.yaml

                        sed -i \
                          "s|retail-store-cart:.*|retail-store-cart:${IMAGE_TAG}|g" \
                          environments/dev/deploy.yaml

                        sed -i \
                          "s|retail-store-orders:.*|retail-store-orders:${IMAGE_TAG}|g" \
                          environments/dev/deploy.yaml

                        sed -i \
                          "s|retail-store-checkout:.*|retail-store-checkout:${IMAGE_TAG}|g" \
                          environments/dev/deploy.yaml

                        git add environments/dev/deploy.yaml

                        if git diff --cached --quiet; then
                          echo "No GitOps changes found"
                        else
                          git commit \
                            -m "Deploy retail store ${IMAGE_TAG}"

                          git push origin main
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Images pushed with tag: ${IMAGE_TAG}"
            echo "Argo CD will deploy from the GitOps repository."
        }

        failure {
            echo "Pipeline failed."
        }

        always {
            sh '''
                docker logout "${ECR_REGISTRY}" || true
                docker image prune -f || true
            '''
        }
    }
}