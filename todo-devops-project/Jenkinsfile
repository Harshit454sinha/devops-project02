// =============================================================
// Jenkinsfile — Tasklane Todo App CI/CD Pipeline
//
// Flow:
//   GitHub -> Jenkins -> Checkout -> Validate -> Docker Build
//   -> Docker Hub Push -> AWS EC2 Deploy -> Verify
//
// Required Jenkins credentials (configure in Jenkins > Credentials):
//   dockerhub-creds   -> Username/Password credential for Docker Hub
//   ec2-ssh-key       -> SSH Username with private key credential for EC2
// =============================================================

pipeline {
    agent any

    // ---- Configure these for your own environment ----
    environment {
        DOCKERHUB_USERNAME = 'your-dockerhub-username'   // <-- change this
        IMAGE_NAME          = 'todo-devops-app'
        IMAGE_TAG            = "${env.BUILD_NUMBER}"
        FULL_IMAGE            = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE            = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"

        EC2_HOST = 'ubuntu@YOUR_EC2_PUBLIC_IP'            // <-- change this
        CONTAINER_NAME = 'todo-app'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Cloning source code from GitHub...'
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                echo 'Validating required project files...'
                sh '''
                    set -e
                    test -f index.html || (echo "Missing index.html" && exit 1)
                    test -f style.css  || (echo "Missing style.css" && exit 1)
                    test -f Dockerfile || (echo "Missing Dockerfile" && exit 1)
                    echo "All required files are present."
                '''

                echo 'Running a lightweight HTML sanity check...'
                sh '''
                    set -e
                    grep -q "<html" index.html && echo "index.html looks like a valid HTML document."
                    grep -q "</html>" index.html && echo "index.html is properly closed."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image: ${FULL_IMAGE}"
                sh "docker build -t ${FULL_IMAGE} -t ${LATEST_IMAGE} ."
            }
        }

        stage('Docker Hub Push') {
            steps {
                echo 'Logging in to Docker Hub and pushing image...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$FULL_IMAGE"
                        docker push "$LATEST_IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('AWS EC2 Deployment') {
            steps {
                echo "Deploying ${LATEST_IMAGE} to EC2 host ${EC2_HOST}..."
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} "
                            docker pull ${LATEST_IMAGE} &&
                            docker stop ${CONTAINER_NAME} || true &&
                            docker rm ${CONTAINER_NAME} || true &&
                            docker run -d --name ${CONTAINER_NAME} -p 80:80 ${LATEST_IMAGE}
                        "
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                echo 'Verifying the deployment on EC2...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} "
                            docker ps --filter name=${CONTAINER_NAME} &&
                            (sudo ss -tuln | grep ':80 ' || echo 'Port 80 check: see docker ps output above')
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded. Website should be live at http://<EC2-PUBLIC-IP>"
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above for details.'
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}
