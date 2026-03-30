pipeline {
    agent {
        docker {
            image 'google/cloud-sdk:slim'
            args '-u root'
        }
    }
    
    options {
        skipStagesAfterUnstable()
    }

    environment {
        PROJECT_ID = 'id-proyek-gcp-anda'
        ZONE       = 'asia-southeast2-a'
        VM_NAME    = "jenkins-deploy-temp-${BUILD_NUMBER}"
        MACHINE_TYPE = 'e2-medium'
    }

    stages {
        stage('Preparation') {
            steps {
                echo 'Installing dependencies inside Docker container...'
                sh 'pip install pytest pyinstaller'
                withCredentials([file(credentialsId: 'gcp-service-account-key', variable: 'GCP_KEY')]) {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_KEY}
                        gcloud config set project ${PROJECT_ID}
                    """
                }
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

        stage('Manual Approval') {
            steps {
                input message: 'Lanjutkan ke tahap Deploy?', ok: 'Proceed'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    try {
                        echo "1. Membuat Instance GCE: ${VM_NAME}..."
                        sh """
                            gcloud compute instances create ${VM_NAME} \
                                --zone=${ZONE} \
                                --machine-type=${MACHINE_TYPE} \
                                --image-family=ubuntu-2204-lts \
                                --image-project=ubuntu-os-cloud \
                                --tags=http-server,https-server
                        """

                        echo "2. Simulasi Deploy: Aplikasi sedang berjalan di VM baru..."
                        
                        echo "3. Menjeda eksekusi selama 1 menit (Kriteria 3)..."
                        sh 'sleep 60'

                    } finally {
                        echo "4. Menghapus Instance GCE (Kill): ${VM_NAME}..."

                        sh "gcloud compute instances delete ${VM_NAME} --zone=${ZONE} --quiet"
                    }
                }
            }
        }
    }
}