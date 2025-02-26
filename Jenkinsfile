pipeline {
    agent any

    environment {
        GIT_REPO = "git@github.com:livedevops2023/projCert.git"
        DOCKER_IMAGE = "devopsedu/webapp"
        CONTAINER_NAME = "php_web_container"
        TEST_SERVER = "34.239.106.143"
        PUPPET_MASTER = "35.174.154.151"
    }

    stages {
        stage('Install Puppet Agent') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh ubuntu@${TEST_SERVER} "sudo apt update && sudo apt install -y puppet-agent"
                        '''
                    }
                }
            }
        }

        stage('Sign Puppet Certificate') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh ubuntu@${PUPPET_MASTER} "sudo puppet cert sign --all"
                        '''
                    }
                }
            }
        }

        stage('Install Docker via Puppet') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh ubuntu@${TEST_SERVER} "sudo puppet agent --test || true"
                        '''
                    }
                }
            }
        }

        stage('Pull Code and Deploy Container') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh ubuntu@${TEST_SERVER} "rm -rf app && git clone ${GIT_REPO} app"
                        ssh ubuntu@${TEST_SERVER} "cd app && docker build -t ${DOCKER_IMAGE} ."
                        ssh ubuntu@${TEST_SERVER} "docker run -d --name ${CONTAINER_NAME} -p 8080:80 ${DOCKER_IMAGE}"
                        '''
                    }
                }
            }
        }

        stage('Run Selenium Tests') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh ubuntu@${TEST_SERVER} "java -jar /opt/selenium/selenium.jar"
                        '''
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Build failed, deleting container..."
            script {
                sshagent(['slave-cred']) {
                    sh '''
                    ssh ubuntu@${TEST_SERVER} "docker rm -f ${CONTAINER_NAME} || true"
                    '''
                }
            }
        }
    }
}
