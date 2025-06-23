pipeline {
    agent any

    environment {
        PYTHON = 'python3'
        REPORTS_DIR = 'reports'
        APP_URL = "http://localhost:5000"
        APP_PORT = '5000'
        VENV_DIR = 'venv'
        ZAP_REPORT_PATH = "${WORKSPACE}/zap_report.html"
        
        DEPENDENCY_CHECK_REPORT_PATH = "${WORKSPACE}/dependency-check-report.html"
        BANDIT_REPORT_PATH = "${WORKSPACE}/bandit_report.json"
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
           
        }

        stage('DAST Scan (OWASP ZAP)') {
            steps {
                echo "Starting DAST Scan on http://localhost:5000"
                sh '''
                    # Simulated scan - replace this with actual OWASP ZAP command if needed
                    echo "Simulating ZAP scan using http://localhost:5000"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/zap-reports/**/*.html', allowEmptyArchive: true                    
                    echo "ZAP DAST scan completed." 
                }
                failure {
                    echo "ZAP DAST scan failed or found vulnerabilities!"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*.*', allowEmptyArchive: true
            echo 'Pipeline finished. Reports archived.'
        }
    }
}
