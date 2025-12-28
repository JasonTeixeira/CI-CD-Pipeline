pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'dockerhub.io'
        DOCKER_IMAGE = 'your-username/test-app'
        PYTHON_VERSION = '3.10'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse HEAD > .git/commit-id'
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Lint & Format Check') {
            parallel {
                stage('Flake8') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pip install flake8
                            flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
                        '''
                    }
                }
                stage('Black') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pip install black
                            black --check .
                        '''
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest tests/unit -v --junit-xml=reports/unit-tests.xml --cov=. --cov-report=html
                '''
            }
            post {
                always {
                    junit 'reports/unit-tests.xml'
                    publishHTML([
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Code Coverage'
                    ])
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    # Start test database
                    docker-compose -f docker-compose.test.yml up -d
                    sleep 10
                    
                    # Run tests
                    pytest tests/integration -v --junit-xml=reports/integration-tests.xml
                    
                    # Cleanup
                    docker-compose -f docker-compose.test.yml down
                '''
            }
            post {
                always {
                    junit 'reports/integration-tests.xml'
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Bandit') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pip install bandit
                            bandit -r . -f json -o reports/bandit-report.json || true
                        '''
                    }
                }
                stage('Safety') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pip install safety
                            safety check --json > reports/safety-report.json || true
                        '''
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def commitId = readFile('.git/commit-id').trim()
                    docker.build("${DOCKER_IMAGE}:${commitId}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        def commitId = readFile('.git/commit-id').trim()
                        docker.image("${DOCKER_IMAGE}:${commitId}").push()
                        docker.image("${DOCKER_IMAGE}:latest").push()
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    # Update staging environment
                    kubectl set image deployment/app app=${DOCKER_IMAGE}:$(cat .git/commit-id) \
                        --namespace=staging
                    kubectl rollout status deployment/app --namespace=staging
                '''
            }
        }
        
        stage('Smoke Tests') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    . venv/bin/activate
                    pytest tests/smoke --base-url=https://staging.example.com -v
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "Pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Pipeline failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
