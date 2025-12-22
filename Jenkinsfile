pipeline {
    agent { label 'built-in' }

    tools {
        docker 'docker-custom' 
        // Nama 'docker-custom' harus sama persis dengan Langkah 1
    }

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-backend' 
        DOCKER_CREDS = 'docker-hub' 
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS    = 'git-token'
        
        // URL Repo Manifest
        MANIFEST_REPO_URL = 'github.com/diwamln/intern-devops-manifests.git' 
        
        // --- KONFIGURASI PATH MANIFEST (PENTING) ---
        // Path ke file YAML Deployment di repo Manifest
        MANIFEST_TEST_PATH = 'fastapi-backend/dev/deployment.yaml'
        
        // WAJIB ADA: Pastikan kamu sudah buat folder 'prod' dan copy deployment.yaml ke sana
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

        // =========================================
        // FLOW: ENVIRONMENT TESTING
        // =========================================
        stage('Build & Push (TEST Image)') {
            steps {
                script {
                    // FIX: Tidak pakai dir('backend') karena Dockerfile ada di root
                    docker.withRegistry('', DOCKER_CREDS) {
                        def testTag = "${env.BASE_TAG}-test"
                        echo "Building Backend Image: ${testTag}"
                        
                        // DEBUG: Cek isi folder workspace saat ini
                        sh 'ls -la'

                        // Build image dari direktori saat ini (.)
                        def testImage = docker.build("${DOCKER_IMAGE}:${testTag}", ".")
                        testImage.push()
                    }
                }
            }
        }

        stage('Update Manifest (TEST)') {
            steps {
                script {
                    sh 'rm -rf temp_manifests' // Bersihkan workspace sisa build sebelumnya
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            // 1. Clone Repo Manifests
                            sh "git clone https://${GIT_USER}:${GIT_PASS}@${MANIFEST_REPO_URL} ."
                            sh 'git config user.email "jenkins@bot.com"'
                            sh 'git config user.name "Jenkins Pipeline"'
                            
                            // 2. Update Image di YAML Testing
                            // Menggunakan variabel MANIFEST_TEST_PATH yang sudah diset di atas
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' ${MANIFEST_TEST_PATH}"
                            
                            // 3. Push ke Git
                            sh "git add ."
                            sh "git commit -m 'Deploy Backend TEST: ${env.BASE_TAG}-test [skip ci]'"
                            sh "git push origin main"
                        }
                    }
                }
            }
        }

        // =========================================
        // GATE: APPROVAL MANUAL
        // =========================================
        stage('Approval for Production') {
            steps {
                input message: "Backend versi TEST (${env.BASE_TAG}-test) sudah dideploy. Apakah aman untuk lanjut ke PROD?", ok: "Deploy ke Prod!"
            }
        }

        // =========================================
        // FLOW: ENVIRONMENT PRODUCTION
        // =========================================
        stage('Build & Push (PROD Image)') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CREDS) {
                        // Strategi: Retagging (Promote Image) agar efisien
                        // Kita tidak build ulang, tapi mengambil image test yang sudah valid
                        def testImage = docker.image("${DOCKER_IMAGE}:${env.BASE_TAG}-test")
                        def prodTag = "${env.BASE_TAG}-prod"
                        
                        // Pull dulu untuk memastikan image ada di local cache agent
                        testImage.pull() 
                        
                        // Beri tag baru (-prod) dan tag latest
                        testImage.push(prodTag)
                        testImage.push('latest')
                        
                        echo "Image berhasil dipromosikan ke PROD: ${prodTag}"
                    }
                }
            }
        }

        stage('Update Manifest (PROD)') {
            steps {
                script {
                    // Masuk kembali ke folder temp_manifests
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            // Pull perubahan terbaru untuk menghindari konflik git
                            sh "git pull origin main" 
                            
                            // 1. Update Image di YAML Production
                            // Menggunakan variabel MANIFEST_PROD_PATH
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' ${MANIFEST_PROD_PATH}"
                            
                            // 2. Push ke Git
                            sh "git add ."
                            sh "git commit -m 'Promote Backend PROD: ${env.BASE_TAG}-prod [skip ci]'"
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
    }
}
