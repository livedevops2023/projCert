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
        stage('Install Puppet Agent on Slave') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} << 'EOF'
                        sudo apt update
                        sudo apt install -y wget
                        wget https://apt.puppet.com/puppet7-release-jammy.deb
                        sudo dpkg -i puppet7-release-jammy.deb
                        sudo apt update
                        sudo apt install -y puppet-agent
                        echo "server=${PUPPET_MASTER}" | sudo tee -a /etc/puppetlabs/puppet/puppet.conf
                        sudo systemctl restart puppet
                        sudo /opt/puppetlabs/bin/puppet agent --test --waitforcert 60
                        EOF
                        '''
                    }
                }
            }
        }

        stage('Sign Puppet Certificate on Master') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${PUPPET_MASTER} "sudo puppetserver ca sign --all"
                        '''
                    }
                }
            }
        }

        stage('Trigger Puppet Agent to Install Docker') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "sudo puppet agent --test || true"
                        '''
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "rm -rf ~/app && git clone ${GIT_REPO} ~/app"
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "cd ~/app && sudo docker build -t ${DOCKER_IMAGE} ."
                        '''
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "
                        sudo docker stop ${CONTAINER_NAME} || true &&
                        sudo docker rm ${CONTAINER_NAME} || true &&
                        sudo docker run -d -p 8080:80 --name ${CONTAINER_NAME} ${DOCKER_IMAGE}
                        "
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
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "java -jar /opt/selenium/selenium.jar"
                        '''
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Build or tests failed. Cleaning up container..."
            script {
                sshagent(['slave-cred']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "sudo docker rm -f ${CONTAINER_NAME} || true"
                    '''
                }
            }
        }
    }
}
