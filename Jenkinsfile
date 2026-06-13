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
                    python3 --version
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

                    echo "Required files:"
                    test -f README.md
                    test -f Jenkinsfile
                    test -f docs/jenkins-homelab-pipeline.md
                '''
            }
        }

        stage('Validate YAML Files') {
            steps {
                sh '''
                    echo "Searching for YAML files..."

                    YAML_FILES=$(find . -type f \\( -name "*.yaml" -o -name "*.yml" \\) | sort)

                    if [ -z "$YAML_FILES" ]; then
                        echo "No YAML files found. Skipping YAML validation."
                        exit 0
                    fi

                    echo "YAML files found:"
                    echo "$YAML_FILES"

                    python3 - <<'PY'
import sys
from pathlib import Path

try:
    import yaml
except ImportError:
    print("ERROR: PyYAML is not installed on the Jenkins agent.")
    print("Install it with: python3 -m pip install --user pyyaml")
    sys.exit(1)

yaml_files = sorted(
    list(Path(".").rglob("*.yaml")) +
    list(Path(".").rglob("*.yml"))
)

failed = False

for file_path in yaml_files:
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            list(yaml.safe_load_all(f))
        print(f"VALID YAML: {file_path}")
    except Exception as e:
        print(f"INVALID YAML: {file_path}")
        print(f"ERROR: {e}")
        failed = True

if failed:
    sys.exit(1)

print("All YAML files are valid.")
PY
                '''
            }
        }

        stage('Validate Incident Documentation') {
            steps {
                sh '''
                    echo "Checking incident documentation structure..."

                    test -d incident-reports
                    test -d runbooks
                    test -d evidence

                    echo "Incident reports:"
                    find incident-reports -type f -name "*.md" | sort

                    echo "Runbooks:"
                    find runbooks -type f -name "*.md" | sort

                    echo "Evidence files:"
                    find evidence -type f | sort || true
                '''
            }
        }
    }

    post {
        success {
            echo 'Jenkins homelab validation completed successfully.'
        }
        failure {
            echo 'Jenkins homelab validation failed. Check console output.'
        }
    }
}
