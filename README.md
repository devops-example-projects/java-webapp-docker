# java-webapp-docker

Jenkins Docker Swarm Integration
================================

Jenkins Server
===============
sudo su -

# Update
sudo apt update -y

# Install wget unzip
sudo apt-get install wget unzip -y

# Install java
sudo apt install openjdk-8-jdk -y

# Install git
sudo apt-get install git -y

# Install Jenkins
sudo wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

# Install Docker
sudo apt-get update -y

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

vi /etc/sudoers
jenkins ALL=(ALL)   NOPASSWD: ALL
ubuntu  ALL=(ALL)   NOPASSWD: ALL

vi /etc/ssh/sshd_config
PasswordAuthentication Yes
PermitRootLogin Yes

vi /etc/passwd
Add /bash to Jenkins User 

Docker Server
==============
sudo su -

# Update
sudo apt update -y

# Install Docker
sudo apt-get update -y

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl start docker

# Plugins Used in the Project
sshagent
Docker Publish 
Maven
git

# Naming

Maven Tool Name: Maven 3.6.3
DockerHub: username & password
dockerserver: username = ubuntu password = Private Key

Docker Swarm
-------------
# Add User to Docker Group
sudo usermod -aG docker ${USER}

# Initialize Docker Swarm Cluster (From Master)
docker swarm init

# Display cluster nodes in manager machine
docker node ls

# Code

node
{
    def buildNumber = env.BUILD_NUMBER
    def app
    stage("Code Check Out")
    {
        git url: 'https://github.com/MithunTechnologiesDevOps/java-web-app-docker.git',branch: 'master'
    }
    stage("Maven Build")
    {
        def mavenHome = tool name: 'Maven 3.6.3', type: 'maven'
        def mavenCMD = "${mavenHome}/bin/mvn"
        sh "${mavenCMD} clean package"
    }
    stage("Build Docker Image")
    {
        sh "docker build -t adityadevops/java-app-docker-jenkins:${buildNumber} ."
        
        or
        
        app = docker.build("adityadevops/java-app-docker-jenkins:${env.BUILD_NUMBER}")
    }
    stage("Test Image")
    {
        app.inside
        {
            sh 'echo "Test Passed"'
        }
    }
    stage("Push Docker Image")
    {
        withCredentials([string(cerdentialsId: 'docker-pwd', variable: 'DockerHub')])
        {
            sh "docker login -u adityadevops -p ${DockerHub}"
        }
        sh "docker push adityadevops/java-app-docker-jenkins:${buildNumber}"

        or

        docker.withRegistry('https://registry.hub.docker.com','DockerHub')
        {
            app.push()
        }
    }
    stage("Deploy on Docker Swarm Cluster")
    {
        sshagent(['dockerserver'])
        {
            sh "sudo ssh -o StrictHostKeyChecking=no ubuntu@Internal IP docker service rm -f javaapp || true"
            sh "sudo ssh -o StrictHostKeyChecking=no ubuntu@Internal IP docker service create --name javaapp -p 8085:8080 --replicas 2 adityadevops/java-app-docker-jenkins:${env.BUILD_NUMBER} || true"

        }
    }
}

# Docker Swarm cluster using docker-compose 
--------------------------------------------
https://github.com/MithunTechnologiesDevOps/spring-boot-mongo-docker.git

# Code

node
{
    def buildNumber = env.BUILD_NUMBER
    def app
    stage("Code Check Out")
    {
        git url: 'https://github.com/MithunTechnologiesDevOps/spring-boot-mongo-docker.git',branch: 'master'
    }
    stage("Maven Build")
    {
        def mavenHome = tool name: 'Maven 3.6.3', type: 'maven'
        def mavenCMD = "${mavenHome}/bin/mvn"
        sh "${mavenCMD} clean package"
    }
    stage("Build Docker Image")
    {
        sh "docker build -t adityadevops/java-app-docker-jenkins:${buildNumber} ."
        
        or
        
        app = docker.build("adityadevops/java-app-docker-jenkins:${env.BUILD_NUMBER}")
    }
    stage("Test Image")
    {
        app.inside
        {
            sh 'echo "Test Passed"'
        }
    }
    stage("Push Docker Image")
    {
        withCredentials([string(cerdentialsId: 'docker-pwd', variable: 'DockerHub')])
        {
            sh "docker login -u adityadevops -p ${DockerHub}"
        }
        sh "docker push adityadevops/java-app-docker-jenkins:${buildNumber}"

        or

        docker.withRegistry('https://registry.hub.docker.com','DockerHub')
        {
            app.push()
        }
    }
    stage("Remove Local Image")
    {
        sh "docker rmi -f adityadevops/java-app-docker-jenkins"
    }
    stage("Deploy on Docker Swarm Cluster")
    {
        sshagent(['dockerserver'])
        {
            sh 'sudo scp -o StrictHostKeyChecking=no  docker-compose.yml ubuntu@Internal IP:'
            sh 'sudo ssh -o StrictHostKeyChecking=no ubuntu@Internal IP docker stack deploy --prune --compose-file docker-compose.yml springboot'

        }
    }
}

# Docker Stack list
docker stack ls

# Docker Service list
docker service ls

# Docker Stack Remove
docker stack rm springboot
