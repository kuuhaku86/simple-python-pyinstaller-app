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
        PROJECT_ID = 'fine-mile-491813-n4'
        ZONE       = 'asia-southeast2-a'
        VM_NAME    = "jenkins-deploy-temp-${BUILD_NUMBER}"
        MACHINE_TYPE = 'e2-micro'
    }

    stages {
        stage('Preparation') {
            steps {
                echo 'Installing dependencies inside Docker container...'
                sh 'pip install pytest pyinstaller --break-system-packages'

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
                sh 'python3 -m PyInstaller --onefile sources/add2vals.py'
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
                                --tags=http-server,https-server \
                                --metadata=startup-script='#!/bin/bash
                                    apt-get update && apt-get install -y python3'
                        """

                        echo "Menunggu VM siap..."
                        sh 'sleep 20'

                        echo "2. Mengirim file ke VM GCE..."
                        sh "gcloud compute scp dist/add2vals ${VM_NAME}:~/ --zone=${ZONE} --quiet"

                        echo "3. Menjalankan aplikasi di VM GCE..."
                        sh """
                            gcloud compute ssh ${VM_NAME} --zone=${ZONE} --command="chmod +x ~/add2vals && ./add2vals" --quiet
                        """
                        
                        echo "4. Menjeda eksekusi selama 1 menit (Kriteria 3)..."
                        sh 'sleep 60'

                    } finally {
                        echo "5. Menghapus Instance GCE (Kill): ${VM_NAME}..."
                        sh "gcloud compute instances delete ${VM_NAME} --zone=${ZONE} --quiet"
                    }
                }
            }
        }
    }
}