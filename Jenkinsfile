pipeline {
    agent {
        kubernetes {
            // Kita definisikan spec Pod langsung di sini (Infrastructure as Code)
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24.0.6-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: dind-storage
      mountPath: /var/lib/docker
  - name: jnlp
    image: jenkins/inbound-agent:latest
  volumes:
  - name: dind-storage
    emptyDir: {}
'''
        }
    }

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-backend'
        DOCKER_CREDS = 'docker-hub' // Pastikan ID Credentials ini ada di Jenkins
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS = 'git-token' // Pastikan ID Credentials ini ada di Jenkins
        // Tambahkan https:// agar git tidak error
        MANIFEST_REPO_URL = 'https://github.com/diwamln/intern-devops-manifests.git'
        
        // --- KONFIGURASI PATH MANIFEST ---
        MANIFEST_TEST_PATH = 'fastapi-backend/dev/deployment.yaml'
        MANIFEST_PROD_PATH = 'fastapi-backend/prod/deployment.yaml'
    }

    stages {
        stage('Checkout & Versioning') {
            steps {
                checkout scm
                script {
                    // Membuat Tag Unik: build-NOMOR-HASH
                    def commitHash = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    env.BASE_TAG = "build-${BUILD_NUMBER}-${commitHash}"
                    currentBuild.displayName = "#${BUILD_NUMBER} Backend (${env.BASE_TAG})"
                }
            }
        }

        stage('Build & Push Docker') {
            steps {
                // Kita gunakan container 'docker' yang didefinisikan di atas
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                        sh "docker build -t ${DOCKER_IMAGE}:${env.BASE_TAG} ."
                        sh "docker push ${DOCKER_IMAGE}:${env.BASE_TAG}"
                        sh "docker tag ${DOCKER_IMAGE}:${env.BASE_TAG} ${DOCKER_IMAGE}:latest"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Update Manifest & Push') {
            steps {
                // Proses update manifest untuk CD (ArgoCD akan auto-sync setelah ini)
                withCredentials([usernamePassword(credentialsId: "${GIT_CREDS}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git config --global user.email "jenkins@naratel.net.id"
                        git config --global user.name "Jenkins CI/CD"
                        
                        # Clone repo manifest
                        git clone ${MANIFEST_REPO_URL} temp_manifest
                        cd temp_manifest
                        
                        # Update tag di file deployment.yaml (menggunakan sed)
                        sed -i "s|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}|g" ${MANIFEST_TEST_PATH}
                        
                        # Push kembali ke GitHub
                        git add .
                        git commit -m "chore: update backend image to ${env.BASE_TAG}"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/diwamln/intern-devops-manifests.git main
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs() // Bersihkan workspace agar tidak memenuhi disk worker node
        }
    }
}
