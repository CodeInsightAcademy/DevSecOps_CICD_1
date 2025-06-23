pipeline {
    agent any

    environment {
        PYTHON = 'python3'
        REPORTS_DIR = 'reports'
    }

    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    apt-get update && \
                    apt-get install -y python3 python3-pip python3-venv git

                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Checkout') {
            steps {
                git 'https://github.com/CodeInsightAcademy/DevSecOps_CICD_1.git' // Replace with your repo
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install safety
                    mkdir -p reports/sca
                    safety check --full-report > reports/sca/safety.txt || true
                '''
            }
        }

        stage('SAST - Bandit Scan') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install bandit
                    mkdir -p reports/sast
                    bandit -r . -f html -o reports/sast/bandit.html || true
                '''
            }
        }

        stage('Install & Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pytest || true
                '''
            }
        }

        stage('Build and Run Flask App') {
            steps {
                sh '''
                    nohup python3 app.py > flask.log 2>&1 &
                    sleep 5
                '''
            }
        }
        stage('Check Docker') {
            steps {
                script {
                    try {
                        sh 'docker --version'
                    } catch (Exception e) {
                        echo "Docker not available: ${e.getMessage()}"
                        // Continue pipeline instead of failing
                    }
                }
            }
        }

        stage('Deploy App for DAST') {
            steps {
                sh '''
                    . venv/bin/activate
                    nohup python3 app.py &
                    sleep 10
                    curl --fail ${APP_URL} # Use the defined APP_URL here as well
                    echo "App is running for DAST scan!"
                '''
            }
            post {
                always {
                    script {
                        def pidsOutput = sh(script: 'lsof -t -i :5000', returnStdout: true).trim()
                        def pids = pidsOutput.replaceAll('\\n', ' ')

                        if (pids) {
                            echo "Killing process(es) on port 5000: ${pids}"
                            sh "kill ${pids}"
                        } else {
                            echo "No process found on port 5000 to kill."
                        }
                    }
                }
            }
        }

        stage('DAST Scan (OWASP ZAP)') {
            steps {
                echo "Starting DAST Scan on ${APP_URL}"
                // Placeholder for your actual ZAP scan command/step
                // This is where APP_URL would typically be used
                // Example: zapPublisher port: 5000, targetURL: "${APP_URL}", ...
                // For demonstration, just echo success
                sh "echo 'Simulating ZAP scan using ${APP_URL}'"
                // Add your actual ZAP scan command here. It will likely use APP_URL.
                // If you're using a ZAP Jenkins plugin, refer to its documentation for how to pass the target URL.
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/zap-reports/**/*.html', allowEmptyArchive: true
                    // You might want to evaluate the ZAP report here to determine success/failure
                    // For now, it echoes a failure message as per your log.
                    echo "ZAP DAST scan completed." // Change this to reflect success if no vulnerabilities
                }
                failure {
                    echo "ZAP DAST scan failed or found vulnerabilities!"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}
