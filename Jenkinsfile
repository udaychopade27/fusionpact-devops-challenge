pipeline {
    agent any

    parameters {
        choice(
            choices: ['Build_all', 'Build_backend', 'Build_frontend', 'Restart_frontend', 'Restart_backend', 'Restart_all','Stop_all'],
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
        stage('Checkout') {
            steps {
                sh "git --version"
                sh "docker -v"
                sh "echo Branch name: ${GIT_BRANCH}"
            }
        }

        stage('Build_stage') {
            parallel {
                stage('Build_frontend') {
                    when {
                        expression { params.build == 'Build_frontend' || params.build == 'Build_all' }
                    }
                    steps {
                        script {
                                sh '''
                                # Run docker-compose  at build time
                                docker-compose -f docker-compose.yml -f docker.yml build --no-cache frontend
                           '''
                        }
                    }
                }

                stage('Build_backend') {
                    when {
                        expression { params.build == 'Build_backend' || params.build == 'Build_all' }
                    }
                    steps {
                        script {
                                sh '''
                                # Run docker-compose with env file passed at build time
                                docker-compose -f docker-compose.yml -f docker.yml build --no-cache backend
                                '''                           
                        }
                    }
                }
            }
        }

        stage('push_images_to_dockerhub') {
            parallel {
                stage('saving_frontend') {
                    when {
                        expression { params.build == 'Build_frontend' || params.build == 'Build_all' }
                    }
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                            sh """
                                echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                                docker push uday27/fusionpact-frontend:v1.0
                                docker logout
                            """
                        }
                    }
                }

                stage('saving_backend') {
                    when {
                        expression { params.build == 'Build_backend' || params.build == 'Build_all' }
                    }
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                            sh """
                                echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                                docker push uday27/fusionpact-backend:v1.0
                                docker logout
                            """
                        }
                    }
                }
            }
        }

        stage('Transfering_tar_file') {
            steps {
                sshagent(["ec2-ssh-key"]) {
                    sh '''
                        scp -o StrictHostKeyChecking=no docker-compose.yml $REMOTE_USER@$REMOTE_HOST:$RWD/
                        scp -o StrictHostKeyChecking=no docker.yml $REMOTE_USER@$REMOTE_HOST:$RWD/
                    '''
                }
        }

        stage('loading_images') {
            parallel {
                stage('loading_frontend') {
                    when {
                        expression { params.build == 'Build_frontend' || params.build == 'Build_all' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker pull uday27/fusionpact-frontend:v1.0'"
                        }
                    }
                }

                stage('loading_backend') {
                    when {
                        expression { params.build == 'Build_backend' || params.build == 'Build_all' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker pull uday27/fusionpact-backend:v1.0'"
                        }
                    }
                }
            }
        }

        stage('Deploying') {
            parallel {
                stage('Deploy_frontend') {
                    when {
                        expression { params.build == 'Build_frontend' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact up -d frontend'"
                        }
                    }
                }

                stage('Deploy_backend') {
                    when {
                        expression { params.build == 'Build_backend' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact up -d backend'"
                        }
                    }
                }

                stage('Deploy_all') {
                    when {
                        expression { params.build == 'Build_all' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact down'"
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact up -d'"
                        }
                    }
                }
            }
        }

        stage('Restart Containers') {
            parallel {
                stage('Restart_frontend') {
                    when {
                        expression { params.build == 'Restart_frontend' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact restart frontend'"
                        }
                    }
                }

                stage('Restart_backend') {
                    when {
                        expression { params.build == 'Restart_backend' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact restart backend'"
                        }
                    }
                }

                stage('Restart_all') {
                    when {
                        expression { params.build == 'Restart_all' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact down'"
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact up -d'"
                        }
                    }
                }
                stage('Stop_all') {
                    when {
                        expression { params.build == 'Stop_all' }
                    }
                    steps {
                        sshagent(["ec2-ssh-key"]) {
                            sh "ssh ${REMOTE_USER}@${REMOTE_HOST} 'cd ${RWD} && docker-compose -f docker-compose.yml -p fusionpact down'"
                        }
                    }
                }
            }
        }
    }
}