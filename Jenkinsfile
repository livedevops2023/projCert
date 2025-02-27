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
        stage('Install & Configure Puppet on Master') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${PUPPET_MASTER} <<EOF
                        set -e  # Exit on error

                        # Remove old Puppet versions
                        sudo apt remove --purge puppet puppet-agent puppetserver -y || true
                        sudo rm -rf /etc/puppetlabs /var/lib/puppet /var/cache/puppet
                        sudo apt autoremove -y

                        # Install Puppet Server and Puppet Agent
                        wget https://apt.puppet.com/puppet7-release-jammy.deb
                        sudo dpkg -i puppet7-release-jammy.deb
                        sudo apt update
                        sudo apt install -y puppetserver puppet-agent

                        # Configure Puppet Server
                        sudo bash -c 'echo "[server]" > /etc/puppetlabs/puppet/puppet.conf'
                        sudo bash -c 'echo "certname = ${PUPPET_MASTER}" >> /etc/puppetlabs/puppet/puppet.conf'
                        sudo bash -c 'echo "dns_alt_names = puppet, ${PUPPET_MASTER}" >> /etc/puppetlabs/puppet/puppet.conf'

                        # Reduce memory usage if necessary
                        sudo sed -i 's/Xms2g/Xms512m/' /etc/default/puppetserver
                        sudo sed -i 's/Xmx2g/Xmx512m/' /etc/default/puppetserver

                        # Restart Puppet Server
                        sudo systemctl restart puppetserver
                        sudo systemctl enable puppetserver

                        # Allow Puppet communication on port 8140
                        sudo ufw allow 8140/tcp || true
                        sudo ufw reload || true
EOF
                        '''
                    }
                }
            }
        }

        stage('Install & Configure Puppet Agent on Slave') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} <<EOF
                        set -e

                        # Remove old Puppet versions
                        sudo apt remove --purge puppet puppet-agent -y || true
                        sudo rm -rf /etc/puppetlabs /var/lib/puppet /var/cache/puppet
                        sudo apt autoremove -y

                        # Install Puppet Agent
                        wget https://apt.puppet.com/puppet7-release-jammy.deb
                        sudo dpkg -i puppet7-release-jammy.deb
                        sudo apt update
                        sudo apt install -y puppet-agent

                        # Configure Puppet Agent to talk to Puppet Master
                        sudo bash -c 'echo "[agent]" > /etc/puppetlabs/puppet/puppet.conf'
                        sudo bash -c 'echo "server = ${PUPPET_MASTER}" >> /etc/puppetlabs/puppet/puppet.conf'

                        # Restart Puppet Agent
                        sudo systemctl restart puppet
                        sudo systemctl enable puppet

                        # Request Puppet Certificate
                        sudo /opt/puppetlabs/bin/puppet agent --test --waitforcert 60
EOF
                        '''
                    }
                }
            }
        }

        stage('Approve Puppet Certificates') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${PUPPET_MASTER} "sudo /opt/puppetlabs/bin/puppetserver ca sign --all"
                        '''
                    }
                }
            }
        }

        stage('Trigger Puppet Agent to Apply Configuration on Slave') {
            steps {
                script {
                    sshagent(['slave-cred']) {
                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER} "sudo /opt/puppetlabs/bin/puppet agent --test"
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
