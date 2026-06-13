pipeline {
    agent { label 'homelab' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Homelab Agent') {
            steps {
                sh '''
                    echo "Running on Jenkins homelab agent"
                    hostname
                    whoami
                    pwd

                    echo "Tool versions"
                    java -version
                    git --version
                    docker --version || true
                    kubectl version --client || true
                '''
            }
        }

        stage('Validate Repository Structure') {
            steps {
                sh '''
                    echo "Repository root files:"
                    ls -la

                    echo "Markdown files:"
                    find . -maxdepth 4 -name "*.md" | sort

                    echo "Incident folders/files:"
                    find . -maxdepth 4 -type f | sort
                '''
            }
        }

        stage('Basic Documentation Check') {
            steps {
                sh '''
                    echo "Checking README exists..."
                    test -f README.md

                    echo "Checking Jenkinsfile exists..."
                    test -f Jenkinsfile

                    echo "Checking for markdown documentation..."
                    find . -name "*.md" | grep -q .
                '''
            }
        }
    }

    post {
        success {
            echo 'Jenkins homelab pipeline validation completed successfully.'
        }
        failure {
            echo 'Jenkins homelab pipeline validation failed. Check console output.'
        }
    }
}
