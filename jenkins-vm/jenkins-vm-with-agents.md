# DATE: 28-09-2023
## RESULT:
- SUCCESS! After setting up SSH: ssh login from controller to agent and vice-versa
- Build runs there

## REFERENCES:
- https://docs.dman.cloud/tutorial-documentation/install-jenkin
- https://www.youtube.com/watch?v=q4g7KJdFSn0&t=2726s
    - https://github.com/dmancloud/complete-prodcution-e2e-pipeline
    - https://github.com/dmancloud/gitops-complete-prodcution-e2e-pipeline

# Az Login
    az login --tenant pietronromanolive.onmicrosoft.com

# Controller VM
## Variables
    rg=jenkins-vm-rg
    vm_jenkins=jenkins-vm
    user=jenkins
    loc=westeurope

## Run az group create to create a resource group.
    az group create --name $rg --location westeurope

## Create Jenkins VM: need pwd initially to setup ssh between vms
    az vm create \
    --resource-group $rg \
    --name $vm_jenkins \
    --image Ubuntu2204 \
    --admin-username $user \
    --admin-password Axml-xsl0123 \
    --public-ip-sku Standard
  
## Enable auto-shutdown
    az vm auto-shutdown -g $rg -n $vm_jenkins --time 2300 --email "pietronromano@live.com" 


## As Jenkins runs on port 8080, run az vm open to open port 8080 on the new virtual machine.
    az vm open-port \
    --resource-group $rg \
    --name $vm_jenkins  \
    --port 8080 --priority 1010

## Jenkins agents connect to the controller via port 50000, ensure that port is allowed inbound to the Jenkins Controller.
    az vm open-port \
    --resource-group $rg \
    --name $vm_jenkins  \
    --port 50000 --priority 1050

## Get Controller IP
  ip="$(az vm show \
    --resource-group $rg \
    --name $vm_jenkins -d \
    --query [publicIps] \
    --output tsv)"
  echo $ip

## Log in to the virtual machine using an SSH tool.
    ssh $user@$ip

## Install the JDK.
### Become root, have previously logged on as jenkins
    sudo -i

## Update packages
    apt update
    apt upgrade

### Add adoptium repository
  wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
  echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list

### Update repository and install Java
    apt update
    apt install temurin-17-jdk
### Check version, java is in the path (next two are the same)
    java --version
    /usr/bin/java --version

## Install Jenkins
### First, add the repository key to the system:
    curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
      /usr/share/keyrings/jenkins-keyring.asc > /dev/null
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
      https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
      /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install jenkins

### Starting Jenkins: using systemctl:
    systemctl start jenkins

### Since systemctl doesn’t display status output, we’ll use the status command to verify that Jenkins started successfully:
    systemctl status jenkins
### If everything went well, the beginning of the status output shows that the service is active and configured to start at boot:
    Output
    ● jenkins.service - LSB: Start Jenkins at boot time
      Loaded: loaded (/etc/init.d/jenkins; generated)
      Active: active (exited) since Fri 2020-06-05 21:21:46 UTC; 45s ago
        Docs: man:systemd-sysv-generator(8)
        Tasks: 0 (limit: 1137)
      CGroup: /system.slice/jenkins.service

### NOTE: this also outputs the initial password
    23c8f043d1774faab7094306c7b8a33e
### Also find at
    /var/lib/jenkins/secrets/initialAdminPassword

### Access Jenkins User Interface
http://52.166.179.138:8080
### User: pietronromano, Travies1

### Example Pipeline: SEE Pipeline-HelloWorld.md 

# Agent  VM (user another bash window)
## Create VM
    vm_agent=agent_vm
    az vm create \
    --resource-group $rg \
    --name $vm_agent \
    --image Ubuntu2204 \
    --admin-password Axml-xsl0123 \
    --admin-username $user \
    --public-ip-sku Standard
  
### Enable auto-shutdown
    az vm auto-shutdown -g $rg -n $vm_agent --time 2300 --email "pietronromano@live.com" 

### IP
  ip="$(az vm show \
    --resource-group $rg \
    --name $vm_agent -d \
    --query [publicIps] \
    --output tsv)"
  echo $ip
  13.95.168.237

## Log in to the virtual machine using an SSH tool.
    ssh $user@$ip

## NOT NEEDED: Create Jenkins User (I already added this on creation)
    sudo adduser jenkins

## Grant Sudo Rights to Jenkins User
    sudo usermod -aG sudo jenkins

## Become root, have previously logged on as jenkins
    sudo -i

## Update packages
    apt update
    apt upgrade

## Add adoptium repository
  wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
  echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list

## Update repository and install Java
    apt update
    apt install temurin-17-jdk
## Check version, java is in the path (next two are the same)
    java --version
    /usr/bin/java --version

## Docker
    apt-get update

    apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release

### Add Docker’s official GPG key:
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

## Use the following command to set up the repository:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

## Install Docker Engine
    apt-get update
    apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

### NOT NEEDED: Manage Docker as a non-root user: Create the docker group.
    sudo groupadd docker

### Add your jenkins user to the docker group.
    su jenkins
    sudo usermod -aG docker $USER
    newgrp docker

## Verify that you can run docker commands without sudo.
    docker run hello-world


# Connect to Remote SSH Agent from the Jenkins Controller
## (Move to previous bash session for jenkins-vm)
## Connect to agent (NOTE: INITIALLY REQUIRES A PASSWORD FOR THE jenkins user)
    exit
    agent_ip=13.95.168.237
    ssh jenkins@$agent_ip

## Create private and public SSH keys. The following command creates the private key jenkinsAgent_rsa and the public key jenkinsAgent_rsa.pub. It is recommended to store your keys under ~/.ssh/ so we move to that directory before creating the key pair.
    mkdir ~/.ssh; cd ~/.ssh/ && ssh-keygen -t rsa -m PEM -C "Jenkins agent key" -f "jenkinsAgent_rsa"
## Add the public SSH key to the list of authorized keys on the agent machine
    cat jenkinsAgent_rsa.pub >> ~/.ssh/authorized_keys

## Ensure that the permissions of the ~/.ssh directory is secure, as most ssh daemons will refuse to use keys that have file permissions that are considered insecure:
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys ~/.ssh/jenkinsAgent_rsa*

## Copy the private SSH key (~/.ssh/jenkinsAgent_rsa) from the agent machine to your OS clipboard
    cat ~/.ssh/jenkinsAgent_rsa

## From the agent machine, ssh to the controller, to add it to known hosts
    ssh jenkins@52.166.179.138

# Add the Agent on the Jenkins UI (Controller)
- Disable built in node by setting executors to 0
- New Node: 
  - jenkins-agent
  - /home/jenkins
  - user: jenkins
  - KEy: paste in private key, then select the credential

  ## Run the HelloWorld pipeline again - it should run on the agent

# Install Plugins and Tools in Jenkins Controller
- Plugins
    - Maven Integration, Pipeline Maven Integration
    - Eclipse Temurin installer
- Tools
    - Maven3: Install Automatically
    - Java17: from adoptium.net, leave "JAVA_HOME" blank, Install Automatically, v 17.0.8.1+

# Production e2e app
## Fork the repo to github account
- https://github.com/dmancloud/complete-prodcution-e2e-pipeline.git
- https://github.com/pietronromano/complete-prodcution-e2e-pipeline

## Create a Github Personal Access Token
- https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token
 - Settings -> Developer Settings -> Personal Access Tokens

    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

## Add Credentials in Jenkins
- Credentials -> "Stores Scoped to Jenkins" -> global (dropdown) -> Add Credentials
  - Username with password: 
    - username: pietronromano
    - password: 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
    - id: github

## Create pipeline
- Use SCM: https://github.com/pietronromano/complete-prodcution-e2e-pipeline
- Jenkinsfile.basic
- branch: main

## SUCCESS!
- jenkins@agentvm:~/workspace/e2e-production/target$ java -jar demoapp.jar
