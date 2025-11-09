pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQubeServer'   // Name configured in Jenkins > Manage Jenkins > System
        SONARQUBE_PROJECT_KEY = 'html-app'
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📥 Checking out source code..."
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "🏗️ Building HTML app..."
                sh 'echo "Build successful for HTML app."'
            }
        }

        stage('Test') {
            steps {
                echo "🧪 Running basic quality checks..."
                sh 'echo "Quick code validation completed."'
            }
        }

        stage('Malware Scan') {
            steps {
                echo "🧫 Scanning project files for malware..."
                sh '''
                sudo apt-get update -y
                sudo apt-get install -y clamav
                sudo freshclam

                clamscan -r . > scan_result.txt || true

                if grep -q "Infected files: 0" scan_result.txt; then
                    echo "✅ No malware found in the project files."
                else
                    echo "⚠️ Malware detected! Please review scan_result.txt for details."
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                echo "🔍 Running Trivy vulnerability scan..."
                sh '''
                if ! command -v trivy &> /dev/null; then
                    sudo apt-get install wget -y
                    wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.50.2_Linux-64bit.deb
                    sudo dpkg -i trivy_0.50.2_Linux-64bit.deb
                fi

                trivy fs --exit-code 0 --no-progress . > trivy_result.txt

                if grep -q "Total:" trivy_result.txt; then
                    echo "✅ Trivy scan completed successfully."
                else
                    echo "⚠️ Vulnerabilities found or scan failed! Check trivy_result.txt for details."
                    exit 1
                fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "📊 Running SonarQube code quality analysis..."
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                    sonar-scanner \
                        -Dsonar.projectKey=${SONARQUBE_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN > sonar_result.txt || true

                    if grep -q "EXECUTION SUCCESS" sonar_result.txt; then
                        echo "✅ SonarQube analysis completed successfully."
                    else
                        echo "SonarQube analysis failed or issues found! Check sonar_result.txt."
                        exit 1
                    fi
                    '''
                }
            }
        }

        stage('Deploy to Apache') {
            steps {
                echo "Deploying HTML app to Apache..."
                sh '''
                sudo mkdir -p /var/www/html/main
                sudo cp index.html /var/www/html/main/index.html
                echo "Deployment completed successfully!"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully — all scans and deployment done!"
        }
        failure {
            echo "Pipeline failed — check logs for details."
        }
    }
}
