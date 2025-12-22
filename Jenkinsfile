pipeline {
    agent any

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-backend' // Nama Image Backend
        DOCKER_CREDS = 'docker-hub' 
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS    = 'git-token'
        
        // URL Repo Manifest
        MANIFEST_REPO_URL = 'github.com/diwamln/intern-devops-manifests.git' 
    }

    stages {
        stage('Checkout & Versioning') {
            steps {
                checkout scm
                script {
                    // Tag Unik: build-NOMOR-HASH
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
                    // Asumsi: Source code backend ada di folder 'backend'
                    dir('backend') { 
                        docker.withRegistry('', DOCKER_CREDS) {
                            def testTag = "${env.BASE_TAG}-test"
                            echo "Building Backend Image: ${testTag}"
                            
                            // Backend tidak butuh --build-arg untuk URL
                            // Config diambil dari K8s ConfigMap nanti
                            def testImage = docker.build("${DOCKER_IMAGE}:${testTag}", ".")
                            testImage.push()
                        }
                    }
                }
            }
        }

        stage('Update Manifest (TEST)') {
            steps {
                script {
                    sh 'rm -rf temp_manifests'
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            // 1. Clone Repo Manifests
                            sh "git clone https://${GIT_USER}:${GIT_PASS}@${MANIFEST_REPO_URL} ."
                            sh 'git config user.email "jenkins@bot.com"'
                            sh 'git config user.name "Jenkins Pipeline"'
                            
                            // 2. Update file YAML Backend Testing
                            // Pastikan strukturnya sesuai (misal: backend/k8s/test.yaml)
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' backend/k8s/test.yaml"
                            
                            // 3. Push
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
                input message: "Backend versi TEST (${env.BASE_TAG}-test) sudah dideploy. DB Migration aman? Lanjut ke PROD?", ok: "Deploy ke Prod!"
            }
        }

        // =========================================
        // FLOW: ENVIRONMENT PRODUCTION
        // =========================================
        stage('Build & Push (PROD Image)') {
            steps {
                script {
                    dir('backend') {
                        docker.withRegistry('', DOCKER_CREDS) {
                            // Strategi: Retagging
                            // Kita tidak perlu build ulang dari nol karena code backend sama saja
                            // Kita ambil image test, lalu kita kasih label 'prod'
                            
                            def testImage = docker.image("${DOCKER_IMAGE}:${env.BASE_TAG}-test")
                            def prodTag = "${env.BASE_TAG}-prod"
                            
                            // Pull dulu untuk memastikan image ada (jika ganti node agent)
                            testImage.pull() 
                            
                            // Beri tag baru (-prod) dan tag latest
                            testImage.push(prodTag)
                            testImage.push('latest')
                            
                            echo "Image berhasil dipromosikan ke PROD: ${prodTag}"
                        }
                    }
                }
            }
        }

        stage('Update Manifest (PROD)') {
            steps {
                script {
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            sh "git pull origin main" // Ambil update terbaru
                            
                            // Update file YAML Backend Production
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' backend/k8s/prod.yaml"
                            
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
