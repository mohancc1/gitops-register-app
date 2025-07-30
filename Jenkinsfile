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
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    script {
                        def imageName = "${DOCKER_USER}/${APP_NAME}"

                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                            def dockerImage = docker.build(imageName)
                            dockerImage.push("${IMAGE_TAG}")
                            dockerImage.push("latest")
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    script {
                        sh """
                            docker run -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:latest \
                            --no-progress --scanners vuln \
                            --exit-code 0 --severity HIGH,CRITICAL --format table
                        """
                    }
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    script {
                        sh "docker rmi ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} || true"
                        sh "docker rmi ${DOCKER_USER}/${APP_NAME}:latest || true"
                    }
                }
            }
        }
    }
}
