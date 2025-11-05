pipeline {
    agent { label 'sonar' }  // Reuse your Sonar agent (must have Node.js installed)

    // No JDK/Maven — not needed for frontend
    // tools { }  // Removed

    environment {
        // App
        APP_NAME        = 'AngularCalculator'
        GIT_REPO        = 'https://github.com/ashuvee/AngularCalculator'
        NGINX_SERVER    = '18.116.203.32'

        // Tools
        SONAR_SERVER    = 'sonar'  // Matches your Jenkins SonarQube config
        NEXUS_URL       = 'http://3.19.221.46:8081'
        NEXUS_REPO      = 'raw'   // Must be a 'raw' hosted repo in Nexus
        VERSION         = "0.0.${BUILD_NUMBER}"
        ZIP_FILE        = "${APP_NAME}-${VERSION}.zip"
    }

    stages {
        /* === Stage 1: Checkout Code === */
        stage('Checkout Code') {
            steps {
                echo '📦 Cloning Angular app from GitHub...'
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }

        /* === Stage 2: SonarQube Analysis === */
        stage('SonarQube Analysis') {
            steps {
                echo '🔍 Running SonarQube static analysis (JavaScript)...'
                withCredentials([string(credentialsId: 'sonar', variable: 'sonar')]) {
                    sh '''
                        npm install -g sonarqube-scanner
                        sonar-scanner \
                          -Dsonar.projectKey=angular-frontend \
                          -Dsonar.projectName="Angular Calculator" \
                          -Dsonar.sources=src \
                          -Dsonar.exclusions=**/node_modules/**,**/*.spec.ts \
                          -Dsonar.host.url=http://YOUR_SONAR_IP:9000 \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        /* === Stage 3: Build Artifact === */
        stage('Build Artifact') {
            steps {
                echo '⚙️ Building Angular app...'
                sh 'export NODE_OPTIONS=--openssl-legacy-provider'
                sh 'npm ci'
                sh 'npm run build -- --configuration=production'
                sh '''
                    cd dist/ai-technov-angularcalculator
                    zip -r "../${ZIP_FILE}" .
                '''
                sh "echo ✅ Build completed! Artifact: ${ZIP_FILE}"
                sh "ls -lh ../${ZIP_FILE}"
            }
        }

        /* === Stage 4: Upload Artifact to Nexus (via REST API) === */
        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USR', passwordVariable: 'NEXUS_PSW')]) {
                    sh '''#!/bin/bash
                        set -e
                        echo "📤 Uploading ${ZIP_FILE} to Nexus..."

                        curl -f -u ${NEXUS_USR}:${NEXUS_PSW} --upload-file "../${ZIP_FILE}" \\
                          "${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${VERSION}/${ZIP_FILE}"

                        echo "✅ Artifact uploaded successfully to Nexus!"
                    '''
                }
            }
        }

        /* === Stage 5: Deploy to NGINX === */
        stage('Deploy to NGINX') {
            // special 'tomcat' agent with also nginx installed
            agent { label 'tomcat' }
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USR', passwordVariable: 'NEXUS_PSW'),
                    sshUserPrivateKey(credentialsId: 'angular-ssh', keyFileVariable: 'SSH_KEY')
                ]) {
                    sh '''#!/bin/bash
                        set -e
                        DOWNLOAD_URL="${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${VERSION}/${ZIP_FILE}"

                        # Setup SSH
                        mkdir -p ~/.ssh
                        cp "$SSH_KEY" ~/.ssh/id_rsa
                        chmod 600 ~/.ssh/id_rsa
                        ssh-keyscan -H ${NGINX_SERVER} >> ~/.ssh/known_hosts

                        # Deploy by pulling from Nexus
                        ssh -i ~/.ssh/id_rsa ubuntu@${NGINX_SERVER} "
                            TMP_DIR=/tmp/angular-deploy-\$\$
                            mkdir -p \"\$TMP_DIR\"
                            cd \"\$TMP_DIR\"

                            curl -u ${NEXUS_USR}:${NEXUS_PSW} -s -f -O ${DOWNLOAD_URL}
                            unzip ${ZIP_FILE}

                            sudo rm -rf /var/www/html/*
                            sudo cp -r ./* /var/www/html/
                            sudo chown -R www-www-data /var/www/html

                            rm -rf \"\$TMP_DIR\"
                        "
                    '''
                }
            }
        }
    }

    post {
        success { echo '🎉 Pipeline completed successfully — Angular app live on NGINX!' }
        failure { echo '❌ Pipeline failed — Check Jenkins logs.' }
    }
}
