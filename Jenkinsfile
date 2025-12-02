// =========================
// GLOBAL LOG WRAPPER
// =========================
def runAndLog(cmd) {

    // First, capture output ALWAYS (stdout + stderr)
    def output = sh(script: "${cmd} 2>&1 | tee _tmp_log.txt", returnStdout: true).trim()

    // Second, capture exit status ONLY
    def status = sh(script: "${cmd}", returnStatus: true)

    // Append to master log file
    writeFile(
        file: "pipeline_logs.txt",
        text: (fileExists("pipeline_logs.txt") ? readFile("pipeline_logs.txt") : "") +
              "\n\n==============================\n" +
              "COMMAND EXECUTED:\n${cmd}\n" +
              "EXIT STATUS: ${status}\n" +
              "------------------------------\n" +
              "OUTPUT:\n${output}\n" +
              "==============================\n"
    )

    return [output, status]
}


// =========================
// MAIN PIPELINE
// =========================
pipeline {
    agent any

    parameters {
        choice(
            choices: [
                'Build_all',
                'Build_backend',
                'Build_frontend',
                'Restart_frontend',
                'Restart_backend',
                'Restart_all',
                'Stop_all'
            ],
            description: 'Build parameter choices',
            name: 'build'
        )
    }

    environment {
        REMOTE_HOST = credentials("ec2-ssh-host")
        REMOTE_USER = credentials("ec2-ssh-user")
        RWD = "/home/ubuntu/fusionpact-devops"
    }

    stages {

        // -----------------------------
        // CHECKOUT + ENV VERIFY
        // -----------------------------
        stage('Checkout') {
            steps {
                script {
                    runAndLog("git --version")
                    runAndLog("docker -v")
                    echo "Branch name: ${GIT_BRANCH}"
                }
            }
        }

        // -----------------------------
        // BUILD STAGES (parallel)
        // -----------------------------
        stage('Build_stage') {
            parallel {

                stage('Build_frontend') {
                    when { expression { params.build in ['Build_frontend', 'Build_all'] } }
                    steps {
                        script {
                            runAndLog("docker-compose -f docker-compose.yml -f docker.yml build --no-cache frontend")
                        }
                    }
                }

                stage('Build_backend') {
                    when { expression { params.build in ['Build_backend', 'Build_all'] } }
                    steps {
                        script {
                            runAndLog("docker-compose -f docker-compose.yml -f docker.yml build --no-cache backend")
                        }
                    }
                }
            }
        }

        // -----------------------------
        // PUSH IMAGES TO DOCKERHUB
        // -----------------------------
        stage('push_images_to_dockerhub') {
            parallel {

                stage('saving_frontend') {
                    when { expression { params.build in ['Build_frontend', 'Build_all'] } }
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                            script {
                                runAndLog("echo \"$DOCKERHUB_PASS\" | docker login -u \"$DOCKERHUB_USER\" --password-stdin")
                                runAndLog("docker push uday27/fusionpact-frontend:v1.0")
                                runAndLog("docker logout")
                            }
                        }
                    }
                }

                stage('saving_backend') {
                    when { expression { params.build in ['Build_backend', 'Build_all'] } }
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                            script {
                                runAndLog("echo \"$DOCKERHUB_PASS\" | docker login -u \"$DOCKERHUB_USER\" --password-stdin")
                                runAndLog("docker push uday27/fusionpact-backend:v1.0")
                                runAndLog("docker logout")
                            }
                        }
                    }
                }
            }
        }

        // -----------------------------
        // TRANSFER FILES TO EC2
        // -----------------------------
        stage('Transfering_tar_file') {
            steps {
                sshagent(["ec2-ssh-key"]) {
                    script {
                        runAndLog("scp -o StrictHostKeyChecking=no docker-compose.yml $REMOTE_USER@$REMOTE_HOST:$RWD/")
                        runAndLog("scp -o StrictHostKeyChecking=no docker.yml $REMOTE_USER@$REMOTE_HOST:$RWD/")
                        runAndLog("scp -o StrictHostKeyChecking=no ./frontend/nginx.conf $REMOTE_USER@$REMOTE_HOST:$RWD/")
                    }
                }
            }
        }

        // -----------------------------
        // PULL IMAGES ON REMOTE SERVER
        // -----------------------------
        stage('loading_images') {
            parallel {

                stage('loading_frontend') {
                    when { expression { params.build in ['Build_frontend', 'Build_all'] } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker pull uday27/fusionpact-frontend:v1.0'")
                            }
                        }
                    }
                }

                stage('loading_backend') {
                    when { expression { params.build in ['Build_backend', 'Build_all'] } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker pull uday27/fusionpact-backend:v1.0'")
                            }
                        }
                    }
                }
            }
        }

        // -----------------------------
        // DEPLOY STAGE
        // -----------------------------
        stage('Deploying') {
            parallel {

                stage('Deploy_frontend') {
                    when { expression { params.build == 'Build_frontend' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -p fusionpact up -d frontend'")
                            }
                        }
                    }
                }

                stage('Deploy_backend') {
                    when { expression { params.build == 'Build_backend' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -p fusionpact up -d backend'")
                            }
                        }
                    }
                }

                stage('Deploy_all') {
                    when { expression { params.build == 'Build_all' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -p fusionpact down'")
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -p fusionpact up -d'")
                            }
                        }
                    }
                }
            }
        }

        // -----------------------------
        // RESTART & STOP
        // -----------------------------
        stage('Restart Containers') {
            parallel {

                stage('Restart_frontend') {
                    when { expression { params.build == 'Restart_frontend' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose restart frontend'")
                            }
                        }
                    }
                }

                stage('Restart_backend') {
                    when { expression { params.build == 'Restart_backend' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose restart backend'")
                            }
                        }
                    }
                }

                stage('Restart_all') {
                    when { expression { params.build == 'Restart_all' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose down'")
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose up -d'")
                            }
                        }
                    }
                }

                stage('Stop_all') {
                    when { expression { params.build == 'Stop_all' } }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            script {
                                runAndLog("ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose down'")
                            }
                        }
                    }
                }
            }
        }
    }

    // -----------------------------
    // POST ACTIONS
    // -----------------------------
    post {
        always {
            emailext(
                to: "udaychopade27@gmail.com",
                subject: "Pipeline Result: ${currentBuild.currentResult}",
                body: "Pipeline completed. Logs are attached.",
                attachmentsPattern: "pipeline_logs.txt"
            )
        }
    }
}
