pipeline {
    agent any
    environment{
        ARTIFACTORY_LOGIN=credentials('artifactory-login')
        SERVER_LOGIN=credentials('linux')
    }
    parameters {
        choice(name: 'BASE_INSTALLATION',
                choices: ['WITH_BASE_INSTALLATION', 'NO'],
                description: 'Choose to install the base installation on the server')
        
        string(name: 'IMAGE',
               defaultValue: 'hello',
               description: 'Docker Image Name')

        string(name: 'ARTIFACTORY_URL',
               defaultValue: 'ptt-docker-local.bin.swisscom.com',
               description: 'Artifactory URL')

        string(name: 'SERVER_URL',
               defaultValue: 'ubuntu@ec2-3-121-186-210.eu-central-1.compute.amazonaws.com',
               description: 'user@server Address for ssh connection')
    }
    
    stages {
        stage('Git checkout') {
            steps {
                echo "Git checkout"
                sh 'git --version'
                git branch: 'main', url: 'https://github.com/5IGE/ci-cd-test.git'
            }
        }
        stage('Base installation') {
            when {
                // Only deploy if the base installation is 'WITH_BASE_INSTALLATION'
                expression { params.BASE_INSTALLATION == 'WITH_BASE_INSTALLATION' }
            }
            steps {
                echo "Base installation on server"
                catchError {
                    sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'sudo apt-get update && sudo apt-get install ca-certificates curl gnupg lsb-release -y'"
                    sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --batch --yes --dearmor -o /etc/apt/keyrings/docker.gpg'"
                    sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'echo \"deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable\" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null'"
                    sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y && sudo chmod 666 /var/run/docker.sock'"
                }
            }
        }
        stage('Build Docker image') {
            steps {
                echo "Build Docker image ${params.IMAGE}"
                sh "docker build -t ${params.ARTIFACTORY_URL}/${params.IMAGE} ."
                sh "docker prune"
                sh "docker run ${params.ARTIFACTORY_URL}/${params.IMAGE}"
            }
        }
        stage('Push Docker image to jFrog') {
            steps {
                echo "Push Docker image to jFrog"
                sh "docker login ${params.ARTIFACTORY_URL} --username ${env.ARTIFACTORY_LOGIN_USR} --password ${env.ARTIFACTORY_LOGIN_PSW}"
                sh "docker push ${params.ARTIFACTORY_URL}/${params.IMAGE}"
                sh "docker logout ${params.ARTIFACTORY_URL}"
            }
        }
        stage('Deploy on Server') {
            steps {
                echo "Deploy on Server"
                sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'docker login ${params.ARTIFACTORY_URL} --username ${env.ARTIFACTORY_LOGIN_USR} --password ${env.ARTIFACTORY_LOGIN_PSW}'"
                sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'docker pull ${params.ARTIFACTORY_URL}/${params.IMAGE}'"
                sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'docker logout ${params.ARTIFACTORY_URL}'"
            }
        }
        stage('Start on Server') {
            steps {
                echo "Start on Server"
                sh "ssh -o StrictHostKeyChecking=no -i ${env.SERVER_LOGIN} ${params.SERVER_URL} 'docker run --rm ${params.ARTIFACTORY_URL}/${params.IMAGE}'"
            }
        }
        stage('Function Test') {
            steps {
                echo "Function Test"
            }
        }
    }
}
