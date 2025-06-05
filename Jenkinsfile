pipeline {
    agent any

    environment {
        SSH_USER = 'safridi'
        REPO_URL = 'https://github.com/safridi-atsg/pipeline-test'
    }

    stages {
        stage('Parse Commit Message') {
            steps {
                dir('repo') {
                    withCredentials([usernamePassword(credentialsId: 'github-username-password', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        script {
                            // Configure authenticated origin and fetch all branches
                            echo "Hello World";
                            sh '''
                                git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/safridi-atsg/pipeline-test
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
                                case 'pre-prod':
                                    env.SSH_HOST = '10.247.109.79'
                                    env.DEPLOY_PATH = '/root/test-pipeline/pipeline-test'
                                    break
                                default:
                                    return null
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

                        sh(script: """
                            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USER@${env.SSH_HOST} << 'EOF'
                        ${scriptToRun}

                        """, label: 'Running remote deploy')

                    }
                }
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

                echo "Deployment Completed ✅"
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

                echo "Rollback Completed ✅"
            """
        default:
            return null
    }
}
