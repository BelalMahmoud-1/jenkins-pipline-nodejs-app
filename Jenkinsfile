pipeline {
    agent any

    tools {
        nodejs 'node-18'
    }

    environment {
        IMAGE_NAME = "belalmahmoud81/react-app"
        // Combine image name + build number into one reusable tag
        IMAGE_TAG  = "${IMAGE_NAME}:${BUILD_NUMBER}"
    }

    stages {

        stage('Install Dependencies') {
            steps {
                // Install all packages from package.json
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                // test the code
                sh 'npm test'
            }
        }

        stage('Build') {
            steps {
                // Compile React app into the /build folder
                sh 'npm run build'
            }
        }

        stage('Docker Build') {
            steps {
                // Build Docker image and tag it with the current build number
                sh "docker build -t ${IMAGE_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                // Inject DockerHub credentials securely — never hardcode passwords
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        #Login using stdin to avoid password showing in logs
                        echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin

                        # Tag with :latest in addition to the build number
                        docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest

                        # Push both tags to DockerHub
                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    # Remove old container if it exists, ignore error if not found
                    docker rm -f react-app || true

                    # Run new container — port 3000:80 avoids needing root for port 80
                    docker run -d --name react-app -p 3000:80 ${IMAGE_TAG}

                """
            }
        }

        stage('Cleanup') {
            steps {
                sh """
                    # Remove versioned image to free disk space
                    # :latest is kept in case a re-deploy is needed
                    docker rmi ${IMAGE_TAG} || true
                """
            }
        }
    }

    post {
        success {
            // Confirm which image was built and pushed
            echo "✅ Pipeline succeeded! Image pushed: ${IMAGE_TAG}"
        }
        failure {
            // Direct link to console logs for faster debugging
            echo "❌ Pipeline failed. Check console: ${BUILD_URL}console"
        }
        always {
            // Save build output as an artifact even if pipeline fails
            archiveArtifacts artifacts: 'build/**', allowEmptyArchive: true

            // Clean workspace to avoid leftover files affecting next build
            cleanWs()
        }
    }
}