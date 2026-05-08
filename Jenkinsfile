pipeline {
    agent any

    environment {
        // SonarQube server name configured in Jenkins:
        // Manage Jenkins -> System -> SonarQube servers
        SONARQUBE_SERVER = 'sonarqube'

        // Docker Hub image name (must be dockerhub_username/repository)
        IMAGE_NAME = 'abd38/python-devops-demo'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-creds',
                    url: 'https://github.com/AB8338/python-devops-demo.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m pip install --upgrade pip
                    pip3 install -r requirements.txt
                    pip3 install pytest pytest-cov
                '''
            }
        }

        stage('Run Tests and Coverage') {
            steps {
                sh '''
                    pytest --cov=. --cov-report xml
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        docker run --rm \
                          --network host \
                          -v "$PWD:/usr/src" \
                          sonarsource/sonar-scanner-cli \
                          -Dsonar.projectKey=python-devops-demo \
                          -Dsonar.projectName=python-devops-demo \
                          -Dsonar.sources=. \
                          -Dsonar.tests=. \
                          -Dsonar.test.inclusions=test_*.py \
                          -Dsonar.python.version=3.8 \
                          -Dsonar.python.coverage.reportPaths=coverage.xml \
                          -Dsonar.exclusions=**/__pycache__/**,**/*.pyc \
                          -Dsonar.analysisCache.enabled=false \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.token=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        /*
        Optional Quality Gate.
        Since the SonarScanner is running inside Docker, Jenkins may not
        always find report-task.txt automatically.
        Uncomment this stage only if report-task.txt is available.

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        */

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Push Docker Image') {
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
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }

        always {
            cleanWs()
        }
    }
}
