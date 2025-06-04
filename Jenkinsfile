pipeline {
    agent any

    environment {
        SSH_USER = 'safridi'
        REPO_URL = 'https://github.com/dinCloud/wtoolgui.git'
    }

    stages {
        stage('Parse Commit Message') {
            steps {
                dir('repo') {
                    withCredentials([usernamePassword(credentialsId: 'github-username-password', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        script {
                            // Configure authenticated origin and fetch all branches
                            sh '''
                                git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dinCloud/wtoolgui.git
                                git fetch --all
                            '''

                            def commitMessage = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
                            echo "Commit message: ${commitMessage}"

                            def deployMatcher = commitMessage =~ /deploy\|([^\|]+)\|([^\|]+)/
                            def rollbackLastMatcher = commitMessage =~ /rollback\|last-hash\|([^\|]+)/
                            def rollbackHashMatcher = commitMessage =~ /rollback\|([a-f0-9]+)\|([^\|]+)/

                            if (deployMatcher) {
                                env.ACTION = 'deploy'
                                env.BRANCH = deployMatcher[0][1]
                                env.SERVER = deployMatcher[0][2]
                            } else if (rollbackLastMatcher) {
                                env.ACTION = 'rollback'
                                env.ROLLBACK_TYPE = 'last'
                                env.SERVER = rollbackLastMatcher[0][1]
                            } else if (rollbackHashMatcher) {
                                env.ACTION = 'rollback'
                                env.ROLLBACK_TYPE = 'exact'
                                env.ROLLBACK_HASH = rollbackHashMatcher[0][1]
                                env.SERVER = rollbackHashMatcher[0][2]
                            } else {
                                error "Invalid commit message format. Expected: deploy|branch|server OR rollback|last-hash|server OR rollback|<hash>|server"
                            }

                            switch (env.SERVER) {
                                case 'staging-nutanix':
                                    env.SSH_HOST = '10.247.108.25'
                                    env.DEPLOY_PATH = '/var/www/dinController-Nutanix/'
                                    break
                                case 'pre-prod':
                                    env.SSH_HOST = '10.247.109.79'
                                    env.DEPLOY_PATH = '/var/www/wtoolgui-docker/wtoolgui/'
                                    break
                                case 'scheduler':
                                    env.SSH_HOST = '10.247.108.31'
                                    env.DEPLOY_PATH = '/var/www/wtoolgui-docker/wtoolgui/'
                                    break
                                case 'heartbeat-daemon':
                                    env.SSH_HOST = '10.247.108.21'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'vmware-daemon':
                                    env.SSH_HOST = '10.247.108.23'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'dinmanage-vip':
                                    env.SSH_HOST = '10.247.109.36'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'dinmanage-1':
                                    env.SSH_HOST = '10.247.109.25'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'dinmanage-2':
                                    env.SSH_HOST = '10.247.109.26'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'dincenter-daemon':
                                    env.SSH_HOST = '10.247.108.20'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'nutanix-daemon':
                                    env.SSH_HOST = '10.247.108.24'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'aws-daemon':
                                    env.SSH_HOST = '10.247.108.22'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'azure-daemon':
                                    env.SSH_HOST = '10.247.108.22'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'nutanix-v3-daemon':
                                    env.SSH_HOST = '10.247.108.27'
                                    env.DEPLOY_PATH = '/'
                                    break
                                case 'vnc-server':
                                    env.SSH_HOST = '10.247.109.51'
                                    env.DEPLOY_PATH = '/'
                                    break
                                default:
                                    error "Unknown server: ${env.SERVER}"
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Remote Server') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key-safridi', keyFileVariable: 'SSH_KEY')]) {
                    script {
                        def isRollback = env.ACTION == 'rollback'
                        def scriptToRun = isRollback
                            ? getRollbackScript(env.SSH_HOST, env.ROLLBACK_TYPE, env.ROLLBACK_HASH)
                            : getDeploymentScript(env.SSH_HOST, env.BRANCH)

                        if (!scriptToRun) {
                            error "No script found for host: ${env.SSH_HOST}"
                        }

                        sh """
                            ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@${env.SSH_HOST} << 'EOF'
                            ${scriptToRun}
                        """
                    }
                }
            }
        }

        stage('Started Cypress') {
            steps {
                sh '''
                    cd /var/www/dinManage-QA-Automation
                    npm run test -- --record --key d75674cb-b872-4a5c-888b-c309d85973e3 --tag "VM Provisioning" --spec "cypress/e2e/pages/Test/*"
                '''
                echo "✅ Cypress tests have completed."
            }
        }
    }
}

def getDeploymentScript(host, branch) {
    switch (host) {
        case '10.247.109.79': // pre-prod
            return """
                sudo su -
                cd ${env.DEPLOY_PATH}
                echo "Staging - Entered project directory"

                git rev-parse HEAD > ~/.last_healthy_commit
                echo "Secured last healthy commit"


                git fetch --all
                git checkout ${branch}
                git pull origin ${branch}
                echo "Updated Git branch: ${branch}"

                docker compose -f docker-compose-stag-prod.yaml build
                docker compose -f docker-compose-stag-prod.yaml up -d
                echo "Docker services started"

                docker image prune -f --filter "dangling=true"
                docker builder prune -f
                echo "Docker cleanup done"
            """
        case '10.247.108.31': // Production
            return """
                sudo su -
                cd ${env.DEPLOY_PATH}
                echo "Production - Entered project directory"

                git rev-parse HEAD > ~/.last_healthy_commit
                echo "Secured last healthy commit"

                git fetch --all
                git checkout ${branch}
                git pull origin ${branch}
                echo "Updated Git branch: ${branch}"

                docker compose -f docker-compose-stag-prod.yaml build
                docker compose -f docker-compose-stag-prod.yaml up -d
                echo "Docker services started"

                docker image prune -f --filter "dangling=true"
                docker builder prune -f
                echo "Docker cleanup done"
            """
        case '10.247.108.25': // Staging Nutanix
            return """
                sudo su -
                su - apache -s /bin/bash
                cd ${env.DEPLOY_PATH}
                echo "Nutanix - Entered project directory"

                git rev-parse HEAD > ~/.last_healthy_commit
                echo "Secured last healthy commit"

                git fetch --all
                git checkout ${branch}
                git pull origin ${branch}
                echo "Updated Git branch: ${branch}"

                python3.10 daemon/w2control.py stop
                python3.10 daemon/w2control.py start
                ps aux | grep python | grep -v grep
                echo "Python daemon restarted"
            """
        case '10.247.108.21': // heartbeat-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.108.23': // vmware-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.109.36': // dinmanage-vip
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.109.25': // dinmanage-1
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.109.26': // dinmanage-2
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.108.20': // dincenter-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.108.24': // nutanix-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.108.22': // aws-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.108.27': // nutanix-v3-daemon
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.109.51': // vnc-server
            return """
                cd ${env.DEPLOY_PATH}
            """
        case '10.247.109.100': // webhvd
            return """
                cd ${env.DEPLOY_PATH}
            """
        default:
            return null
    }
}


def getRollbackScript(host, rollbackType, rollbackHash = "") {
    def hashCommand = rollbackType == 'last' ? "cat ~/.last_healthy_commit" : "echo ${rollbackHash}"

    switch (host) {
        case '10.247.109.79': // pre-prod
            return """
                sudo su -
                cd ${env.DEPLOY_PATH}
                echo "Rollback - Entered project directory"

                git fetch --all
                HASH=\$(${hashCommand})
                git checkout \$HASH
                echo "Rolled back to commit: \$HASH"

                docker compose -f docker-compose-stag-prod.yaml build
                docker compose -f docker-compose-stag-prod.yaml up -d
                echo "Rollback complete"
            """
        case '10.247.108.31': // scheduler
            return """
                sudo su -
                cd ${env.DEPLOY_PATH}
                echo "Rollback - Entered project directory"

                git fetch --all
                HASH=\$(${hashCommand})
                git checkout \$HASH
                echo "Rolled back to commit: \$HASH"

                docker compose -f docker-compose-stag-prod.yaml build
                docker compose -f docker-compose-stag-prod.yaml up -d
                echo "Rollback complete"
            """
        case '10.247.108.25': // staging-nutanix
            return """
                sudo su -
                su - apache -s /bin/bash
                cd ${env.DEPLOY_PATH}
                echo "Rollback - Entered project directory"

                git fetch --all
                HASH=\$(${hashCommand})
                git checkout \$HASH
                echo "Rolled back to commit: \$HASH"

                python3.10 daemon/w2control.py stop
                python3.10 daemon/w2control.py start
                ps aux | grep python | grep -v grep
                echo "Python daemon restarted after rollback"
            """
        case '10.247.108.21': // heartbeat-daemon
        case '10.247.108.23': // vmware-daemon
        case '10.247.109.36': // dinmanage-vip
        case '10.247.109.25': // dinmanage-1
        case '10.247.109.26': // dinmanage-2
        case '10.247.108.20': // dincenter-daemon
        case '10.247.108.24': // nutanix-daemon
        case '10.247.108.22': // aws-daemon / azure-daemon
        case '10.247.108.27': // nutanix-v3-daemon
        case '10.247.109.51': // vnc-server
        case '10.247.109.100': // webhvd
            return """
                sudo su -
                cd ${env.DEPLOY_PATH}
                echo "Rollback - Entered project directory"

                git fetch --all
                HASH=\$(${hashCommand})
                git checkout \$HASH
                echo "Rolled back to commit: \$HASH"
            """
        default:
            return null
    }
}
