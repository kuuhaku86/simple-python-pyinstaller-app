pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-u root'
        }
    }
    
    options {
        skipStagesAfterUnstable()
    }

    stages {
        stage('Preparation') {
            steps {
                echo 'Installing dependencies inside Docker container...'
                sh 'pip install pytest pyinstaller'
            }
        }

        stage('Build') {
            steps {
                sh 'python3 -m py_compile sources/add2vals.py sources/calc.py'
            }
        }

        stage('Test') {
            steps {
                sh 'mkdir -p test-reports'
                sh 'python3 -m pytest --verbose --junit-xml test-reports/results.xml sources/test_calc.py'
            }
            post {
                always {
                    junit 'test-reports/results.xml'
                }
            }
        }

        stage('Deliver') {
            steps {
                sh 'pyinstaller --onefile sources/add2vals.py'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/*', allowEmptyArchive: true
                }
            }
        }
    }
}