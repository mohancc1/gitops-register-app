// Jenkinsfile
pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'java17'
        maven 'Maven3'
    }

    environment {
        SONAR_PROJECT_KEY = 'myapp'
        SONAR_TOKEN       = credentials('jenkins-sonarqube-token') // Ensure this credential ID is correct

        APP_NAME    = "register-app-pipeline"
        RELEASE     = "1.0.0" // Consider deriving this dynamically from your pom.xml for real projects
        IMAGE_TAG   = "${RELEASE}-${BUILD_NUMBER}"

        // IMPORTANT: Define the path to your JAR relative to the Jenkins workspace root.
        // Assuming your 'server' module produces a JAR named 'server-1.0-SNAPSHOT.jar'
        // and your workspace is the root of the 'gitops-register-app' repository.
        JAR_PATH    = "server/target/server-1.0-SNAPSHOT.jar" // <--- VERIFY THIS PATH AND FILENAME!
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs() // Cleans the Jenkins workspace before starting
            }
        }

        stage("Checkout Application Repo") {
            steps {
                // Clones the application repository into the workspace
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/mohancc1/gitops-register-app'
            }
        }

        stage("Build Application") {
            steps {
                // This command builds all modules in your multi-module project.
                // It will ensure the 'server-1.0-SNAPSHOT.jar' is created in server/target/
                sh 'mvn clean package'
            }
        }

        stage("Test Application") {
            steps {
                // Runs unit tests for all modules
                sh 'mvn test'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                        -Dsonar.host.url=http://34.204.170.100:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                    '''

                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'

                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub', // Ensure this credential ID for Docker Hub is correct
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    script {
                        def imageName = "${DOCKER_USER}/${APP_NAME}"

                        // Using the Docker Pipeline plugin's `withRegistry` block for authentication
                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                            // Build the Docker image from the current directory (workspace root)
                            // The Dockerfile MUST be updated to correctly COPY the JAR from `server/target/`
                            def dockerImage = docker.build(imageName, '.')
                            echo "Successfully built Docker image: ${imageName}:${IMAGE_TAG}"

                            // Push the image with the specific tag (e.g., 1.0.0-1)
                            dockerImage.push("${IMAGE_TAG}")
                            echo "Successfully pushed Docker image: ${imageName}:${IMAGE_TAG}"

                            // Push the image with the 'latest' tag
                            dockerImage.push("latest")
                            echo "Successfully pushed Docker image: ${imageName}:latest"
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub', // Use the same Docker Hub credentials for Trivy if needed
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        # Pull the Trivy image and run the scan
                        docker run -v /var/run/docker.sock:/var/run/docker.sock \\
                        aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:latest \\
                        --no-progress --scanners vuln \\
                        --exit-code 0 --severity HIGH,CRITICAL --format table
                        # exit-code 0 means success even if vulnerabilities are found.
                        # Change to --exit-code 1 if you want the pipeline to fail on HIGH/CRITICAL vulnerabilities.
                    """
                }
            }
        }
    }
}
