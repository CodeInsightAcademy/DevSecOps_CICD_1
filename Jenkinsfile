pipeline {
    agent any

    environment {
        PYTHON = 'python3'
        REPORTS_DIR = 'reports'
        APP_URL = "http://localhost:5000"
        APP_PORT = '5000'
        VENV_DIR = 'venv'
        
        // Security report paths
        SAFETY_REPORT = "${REPORTS_DIR}/sca/safety.txt"
        BANDIT_REPORT = "${REPORTS_DIR}/sast/bandit.html"
        ZAP_REPORT = "${REPORTS_DIR}/dast/zap_report.html"
        TEST_REPORT = "${REPORTS_DIR}/tests/results.xml"
    }

    stages {
        stage('Setup Environment') {
            steps {
                script {
                    try {
                        sh """
                            ${PYTHON} -m venv ${VENV_DIR}
                            . ${VENV_DIR}/bin/activate
                            pip install --upgrade pip
                        """
                    } catch (e) {
                        error "Failed to setup Python environment: ${e}"
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh """
                    . ${VENV_DIR}/bin/activate
                    pip install -r requirements.txt
                """
            }
        }

        stage('Security Checks') {
            parallel {
                stage('SCA - Dependency Check') {
                    steps {
                        sh """
                            . ${VENV_DIR}/bin/activate
                            pip install safety
                            mkdir -p ${REPORTS_DIR}/sca
                            safety check --full-report > ${SAFETY_REPORT} || true
                        """
                    }
                }

                stage('SAST - Bandit Scan') {
                    steps {
                        sh """
                            . ${VENV_DIR}/bin/activate
                            pip install bandit
                            mkdir -p ${REPORTS_DIR}/sast
                            bandit -r . -f html -o ${BANDIT_REPORT} || true
                        """
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh """
                    . ${VENV_DIR}/bin/activate
                    mkdir -p ${REPORTS_DIR}/tests
                    pytest --junitxml=${TEST_REPORT} || true
                """
            }
        }

        stage('Deploy for Testing') {
            steps {
                script {
                    try {
                        sh """
                            . ${VENV_DIR}/bin/activate
                            nohup ${PYTHON} app.py > flask.log 2>&1 &
                            sleep 10
                            curl --fail ${APP_URL}
                        """
                    } catch (e) {
                        error "Failed to start application: ${e}"
                    }
                }
            }
        }

        stage('DAST Scan') {
            steps {
                script {
                    try {
                        echo "Starting ZAP scan against ${APP_URL}"
                        // Example using OWASP ZAP Docker image
                        sh """
                            docker run --rm \\
                                -v ${WORKSPACE}/${REPORTS_DIR}/dast:/zap/reports \\
                                -t owasp/zap2docker-stable zap-baseline.py \\
                                -t ${APP_URL} \\
                                -r zap_report.html \\
                                || true
                        """
                    } catch (e) {
                        echo "DAST scan encountered issues: ${e}"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Cleanup running processes
                sh """
                    pkill -f "${PYTHON} app.py" || true
                    docker ps -q --filter ancestor=owasp/zap2docker-stable | xargs -r docker stop || true
                """

                // Archive all reports
                archiveArtifacts artifacts: """
                    ${REPORTS_DIR}/**/*,
                    flask.log
                """, allowEmptyArchive: true

                // Publish test results
                junit allowEmptyResults: true, testResults: "${TEST_REPORT}"
            }
        }

        success {
            slackSend color: 'good', message: "Pipeline SUCCESSFUL: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }

        failure {
            slackSend color: 'danger', message: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }

        unstable {
            slackSend color: 'warning', message: "Pipeline UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}