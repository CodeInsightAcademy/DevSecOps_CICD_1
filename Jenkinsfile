pipeline {
    agent any

    environment {
        PYTHON = 'python3'
        REPORTS_DIR = 'reports'
        APP_URL = "http://localhost:5000"
        APP_PORT = '5000'
        VENV_DIR = 'venv'
        ZAP_REPORT_PATH = "${WORKSPACE}/zap_report.html"

        FLASK_APP = "app.py"
        
        DEPENDENCY_CHECK_REPORT_PATH = "${WORKSPACE}/dependency-check-report.html"
        BANDIT_REPORT_PATH = "${WORKSPACE}/bandit_report.json"

        APP_IMAGE = 'flask-app'
        CONTAINER_NAME = 'flask-app-container'       
        TARGET_URL = "http://localhost:${APP_PORT}"
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

        // stage('Build and Run Flask App') {
        //     steps {
        //         sh '''
        //             nohup python3 app.py > flask.log 2>&1 &
        //             sleep 5
        //         '''
        //     }
        // }
       
        stage('Deploy App Locally') {
            steps {
                // Stop any existing gunicorn process if running on port 5000
                sh '''
                    pkill -f "gunicorn" || true
                '''
                // Start the app with gunicorn in the background, binding to port 5000
                sh '''
                    . ${VENV_DIR}/bin/activate
                    nohup gunicorn --bind 0.0.0.0:5000 app:app > app.log 2>&1 &
                '''
            }
        }
    

        stage('DAST Scan (ZAP)') {
            steps {
                sh '''
                    docker run --rm -v $PWD:/zap/wrk --network host ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py -t ${TARGET_URL} -r zap-report.html || true
                '''
            }
        }

        
    }
     post {
        always {
             sh '''
                pkill -f "gunicorn" || true
            '''
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            archiveArtifacts artifacts: 'reports/**/*.*', allowEmptyArchive: true
            echo 'All reports archived and processes cleaned up.'
        }
    }

    // post {
    //     always {
    //         archiveArtifacts artifacts: 'reports/**/*.*', allowEmptyArchive: true
    //         echo 'Pipeline finished. Reports archived.'
    //     }
    // }
}
