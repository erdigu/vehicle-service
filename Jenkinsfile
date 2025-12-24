pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME          = "erdigvijay/devops_repo:vehicle-service-${BUILD_NUMBER}"
        K8S_NAMESPACE       = "automotive"
        DEPLOYMENT_NAME     = "vehicle-service"
        AWS_DEFAULT_REGION  = "us-east-1"
        // SONAR_TOKEN = credentials('sonar-jenkins-token')
    }

    stages {

        stage('Build JAR (Maven)') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds-4eks',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo "Updating kubeconfig..."
                        sh "aws eks update-kubeconfig --name automotive-cluster --region $AWS_DEFAULT_REGION"

                        echo "Applying Kubernetes manifests..."
                        sh "kubectl apply -f vehicle-service.yaml"

                        echo "Updating deployment image..."
                        sh "kubectl set image deployment/${DEPLOYMENT_NAME} vehicle-service=${IMAGE_NAME} -n ${K8S_NAMESPACE}"

                        echo "Waiting for rollout to complete..."
                        sh "kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${K8S_NAMESPACE}"

                        echo "Deployment successful. Current pods:"
                        sh "kubectl get pods -n ${K8S_NAMESPACE} -l app=vehicle-service"
                    }
                }
            }
        }
    }

    post {
        success {
            emailext(
                subject: "✅ SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:green;">Build Successful 🎉</h2>
                    <p><b>Job:</b> ${JOB_NAME}</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Status:</b> SUCCESS</p>
                    <p><b>Docker Image:</b> ${IMAGE_NAME}</p>
                    <p><b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                """,
                to: "erdigvijaypatil01@gmail.com"
            )
        }

        failure {
            emailext(
                subject: "❌ FAILURE: ${JOB_NAME} #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:red;">Build Failed ❌</h2>
                    <p><b>Job:</b> ${JOB_NAME}</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Status:</b> FAILED</p>
                    <p><b>Console Output:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                """,
                to: "erdigvijaypatil01@gmail.com"
            )
        }

        always {
            sh "docker logout || true"
            sh "docker image prune -f || true"
        }
    }
}
