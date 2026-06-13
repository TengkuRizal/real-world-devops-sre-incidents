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
                    hostname
                    whoami
                    pwd
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
                    ls -la
                    find . -maxdepth 4 -name "*.md" | sort
                    test -f README.md
                '''
            }
        }
    }
}
