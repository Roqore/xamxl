pipeline {
    agent any

    environment {
        DEPLOY_BRANCH = 'origin/main'
        ORG_NAME = 'Roqore'
        REPO_NAME = 'xamxl'
        APP_PATH = "/var/www/${REPO_NAME}"
        HOST = 'xamxl.roqore.com'  // Nginx hostname (used for SSH connection)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "refs/heads/${env.DEPLOY_BRANCH}"]],
                    userRemoteConfigs: [[url: "https://github.com/${env.ORG_NAME}/${env.REPO_NAME}.git"]]
                ])
            }
        }

        stage('Install & Build') {
            when { branch env.DEPLOY_BRANCH }
            steps {
                sh 'npm install'
                sh 'npm run build'
                sh 'npm test'
            }
        }

        stage('Deploy') {
            when { branch env.DEPLOY_BRANCH }
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'do-ssh', keyFileVariable: 'SSH_KEY')
                ]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i \$SSH_KEY root@${env.HOST} '
                            # Install Node.js LTS if not installed
                            if ! command -v node >/dev/null 2>&1; then
                                curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
                                apt-get install -y nodejs build-essential
                            fi

                            # Install PM2 globally if not installed
                            if ! command -v pm2 >/dev/null 2>&1; then
                                npm install -g pm2
                            fi

                            # Create /var/www if missing
                            mkdir -p /var/www

                            # Clone repo if the app path does not exist
                            if [ ! -d "${env.APP_PATH}" ]; then
                                git clone -b ${env.DEPLOY_BRANCH} https://github.com/${env.ORG_NAME}/${env.REPO_NAME}.git ${env.APP_PATH}
                            else
                                cd ${env.APP_PATH}
                                git fetch --all
                                git reset --hard origin/${env.DEPLOY_BRANCH}
                            fi

                            cd ${env.APP_PATH}
                            npm install
                            npm run build

                            # Start or restart app with PM2
                            pm2 restart ${env.REPO_NAME} || pm2 start dist/main.js --name ${env.REPO_NAME}
                        '
                    """
                }
            }
        }
    }

    post {
        success { echo "Deployment successful ✅" }
        failure { echo "Deployment failed ❌" }
    }
}
