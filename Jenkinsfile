pipeline {
    agent {
        docker {
            image 'python:3.9-bullseye' 
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
                echo 'Installing GCloud SDK and Python dependencies...'
                sh '''
                    mkdir -p /usr/share/keyrings
                    
                    apt-get update && apt-get install -y curl gnupg
                    
                    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
                    
                    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee /etc/apt/sources.list.d/google-cloud-sdk.list
                    
                    apt-get update && apt-get install -y google-cloud-cli
                    
                    python3 -m venv venv
                    ./venv/bin/pip install pytest pyinstaller
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                    ./venv/bin/python3 -m py_compile sources/add2vals.py sources/calc.py
                    ./venv/bin/pyinstaller --onefile --paths=sources sources/add2vals.py
                '''
            }
        }

        stage('Test') {
            steps {
                sh 'mkdir -p test-reports'
                sh './venv/bin/python3 -m pytest --verbose --junit-xml test-reports/results.xml sources/test_calc.py'
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
                    withCredentials([file(credentialsId: 'gcp-service-account-key', variable: 'GCP_KEY')]) {
                        try {
                            echo "1. Autentikasi GCP..."
                            sh """
                                gcloud auth activate-service-account --key-file=${GCP_KEY}
                                gcloud config set project ${PROJECT_ID}
                            """

                            echo "2. Membuat Instance GCE: ${VM_NAME}..."
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
                            sh 'sleep 30'

                            echo "3. Mengirim file ke VM GCE..."
                            sh "gcloud compute scp dist/add2vals ${VM_NAME}:~/ --zone=${ZONE} --quiet"

                            echo "4. Menjalankan aplikasi di VM GCE..."
                            sh """
                                gcloud compute ssh ${VM_NAME} --zone=${ZONE} --command="chmod +x ~/add2vals && ./add2vals 10.25 2.5" --quiet
                            """
                            
                            echo "5. Menjeda eksekusi selama 1 menit (Kriteria 3)..."
                            sh 'sleep 60'

                        } finally {
                            echo "6. Menghapus Instance GCE (Kill): ${VM_NAME}..."
                            sh "gcloud compute instances delete ${VM_NAME} --zone=${ZONE} --quiet"
                        }
                    }
                }
            }
        }
    }
}