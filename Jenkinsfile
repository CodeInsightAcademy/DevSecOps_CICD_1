pipeline {
    agent any

    environment {
        PYTHON = 'python3'
        REPORTS_DIR = 'reports'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/CodeInsightAcademy/DevSecOps_CICD_1.git' // Replace with your repo
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    mkdir -p $REPORTS_DIR/sca
                    pip install safety
                    safety check --full-report > $REPORTS_DIR/sca/safety.txt || true
                '''
            }
        }

        stage('SAST - Bandit Scan') {
            steps {
                sh '''
                    mkdir -p $REPORTS_DIR/sast
                    pip install bandit
                    bandit -r . -f html -o $REPORTS_DIR/sast/bandit.html || true
                '''
            }
        }

        stage('Install & Unit Tests') {
            steps {
                sh '''
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

        stage('DAST - ZAP Scan') {
            steps {
                sh '''
                    mkdir -p $REPORTS_DIR/dast
                    docker run -u zap -v $(pwd)/$REPORTS_DIR/dast:/zap/wrk/:rw \
                        -t owasp/zap2docker-stable zap-baseline.py \
                        -t http://localhost:5000 \
                        -r zap-report.html || true
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}
