pipeline {
    agent any

    parameters {
        string(name: 'DEPLOY_URL_CRED_ID', defaultValue: 'DEPLOY_URL', description: 'Credentials ID for the deployment URL (Secret Text)')
        string(name: 'DEPLOY_KEY_CRED_ID', defaultValue: 'DEPLOY_KEY', description: 'Credentials ID for the deployment API key (Secret Text)')
    }

    environment {
        APP_ID = "" // Will be dynamically set
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Env setup') {
            steps {
                sh 'python3 -m venv .venv'
            }
        }

        stage('Install requirements') {
            steps {
                sh '.venv/bin/pip install -r requirements.txt requests'
            }
        }

        stage('Fetch Application ID') {
            steps {
                script {
                    def appId = sh(script: '''.venv/bin/python3 - <<EOF
import requests
import sys

API_URL = "http://13.200.200.238:3000/api/project.all"
api_key = "psWxvPUNiiXpmvwIWMmEJbXMkrubcPbJyDdGHZDVmusuvWAdxWRWyWkmXqCoTdfZ"
app_name = "job-portal-backend-brvs1x"

req = requests.get(API_URL, headers={"accept": "application/json", "x-api-key": api_key}, timeout=30)
req.raise_for_status()
data = req.json()

for project in data or []:
    for env in project.get("environments", []):
        for app in env.get("applications", []):
            if app.get("appName") == app_name:
                print(app.get("applicationId"))
                sys.exit(0)
sys.exit(1)
EOF
''', returnStdout: true).trim()
                    if (!appId) {
                        error "Failed to fetch applicationId!"
                    }
                    env.APP_ID = appId
                    echo "Fetched applicationId: ${env.APP_ID}"
                }
            }
        }

        stage('Migrations') {
            steps {
                sh '.venv/bin/python manage.py makemigrations'
                sh '.venv/bin/python manage.py migrate'
            }
        }

        stage('Unit Tests') {
            steps {
                sh '.venv/bin/python manage.py test'
            }
        }

        stage('Start Server') {
            steps {
                sh '.venv/bin/python manage.py runserver 0.0.0.0:8000 &'
                sh 'sleep 5'
            }
        }

        stage('API Test') {
            steps {
                sh '.venv/bin/python check.py'
            }
        }
    }

    post {
        success {
            echo """
✅
Tests passed, triggering deployment API...
"""
            withCredentials([
                string(credentialsId: params.DEPLOY_URL_CRED_ID, variable: 'DEPLOY_URL'),
                string(credentialsId: params.DEPLOY_KEY_CRED_ID, variable: 'DEPLOY_KEY')
            ]) {
                sh '''
json_payload=$(printf '{"applicationId":"%s"}' "$APP_ID")
curl -fS -X POST "$DEPLOY_URL" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $DEPLOY_KEY" \
  --data-binary "$json_payload" \
  -w "\\nHTTP %{http_code}\\n"
                '''
            }

            mail to: 'xaioene@gmail.com',
                 subject: "Jenkins Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Hello,

The Jenkins pipeline for ${env.JOB_NAME} (build #${env.BUILD_NUMBER}) has succeeded.

* Branch: ${env.BRANCH_NAME}
* Commit: ${env.GIT_COMMIT}
* Build URL: ${env.BUILD_URL}

Deployment API was triggered successfully.

Regards,
Jenkins
"""
        }

        failure {
            echo """
❌
Pipeline failed, sending error email...
"""
            mail to: 'xaioene@gmail.com',
                 subject: "Jenkins Failure: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Hello,

The Jenkins pipeline for ${env.JOB_NAME} (build #${env.BUILD_NUMBER}) has failed.

* Branch: ${env.BRANCH_NAME}
* Commit: ${env.GIT_COMMIT}
* Build URL: ${env.BUILD_URL}

Please check the console output for details.

Regards,
Jenkins
"""
        }
    }
}
