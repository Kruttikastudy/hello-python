pipeline {
     agent { label 'ubuntu-gcp' } // Make sure your Ubuntu VM is configured as a Jenkins agent with this label

    environment {
        SONARQUBE = 'sonarqube'                        // SonarQube server configured in Jenkins
        SCANNER = 'SonarScanner'                        // SonarScanner tool name in Jenkins
        SONAR_PROJECT_KEY = 'hello-python'             // SonarQube project key
        SONAR_API_TOKEN = credentials('sonar-token')   // Jenkins credential for SonarQube token
        SONAR_HOST_URL = 'http://34.47.150.18:9000'   // Jenkins VM IP running SonarQube
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Kruttikastudy/hello-python.git'
            }
        }

        stage('Install & Run Tests') {
            steps {
                sh '''
                  echo "Installing dependencies..."
                  python3 -m pip install --upgrade pip
                  pip3 install --user -r requirements.txt

                  echo "Running tests..."
                  python3 -m pytest -q
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    withEnv(["PATH+SONAR=${tool SCANNER}/bin"]) {
                        sh '''
                          echo "Running SonarQube scan..."
                          sonar-scanner \
                            -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_API_TOKEN \
                            -Dsonar.python.version=3.10
                        '''
                    }
                }
            }
        }

        stage('Deploy to App VM') {
            steps {
                sshagent(credentials: ['gce-ssh']) {
                    sh '''
                        echo "Deploying to App VM..."
                        ssh -o StrictHostKeyChecking=no kruttikahebbar@34.131.33.179 "mkdir -p /home/kruttikahebbar/app"
                        scp -o StrictHostKeyChecking=no -r * kruttikahebbar@34.131.33.179:/home/kruttikahebbar/app/

                        ssh -o StrictHostKeyChecking=no kruttikahebbar@34.131.33.179 "
                          sudo systemctl daemon-reload &&
                          sudo systemctl restart flaskapp &&
                          sudo systemctl enable flaskapp
                        "

                        echo "Deployment complete. Check: curl http://34.131.33.179:8080"
                    '''
                }
            }
        }

        stage('Optional: Check SonarQube Quality Gate') {
            steps {
                script {
                    echo "Fetching SonarQube Quality Gate result (non-blocking)..."
                    sh """
                      curl -s -u $SONAR_API_TOKEN: \
                        "$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=$SONAR_PROJECT_KEY" \
                        | jq '.projectStatus.status'
                    """
                    echo "You can manually inspect the SonarQube dashboard for full details."
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline Succeeded"
        }
        failure {
            echo "Pipeline Failed"
        }
    }
}
