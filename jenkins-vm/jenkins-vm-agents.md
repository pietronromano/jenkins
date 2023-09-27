# DATE: 16-09-2023
# SOURCE: https://learn.microsoft.com/en-us/azure/developer/jenkins/scale-deployments-using-vm-agents?tabs=linux

# RESULT:
- Finally managed to get SSH passwordless auth to work: copied private key and known_hosts to Jenkins server, public key to the Node
- Build with maven on the agent failed: even though installed and can run once logged in
	Caused: java.io.IOException: Cannot run program "mvn" (in directory "/var/lib/jenkins/workspace/Spring - On Agent"): error=2, No such file or directory

# SEE jenkins-get-started for Az login 

## Variables
  rg=jenkins-agent-rg
  vm=jenkins-agent-vm
  user=azureuser
  loc=westeurope

## Run az group create to create a resource group.
  az group create --name $rg --location westeurope

## Create VM
  az vm create \
  --resource-group $rg \
  --name $vm \
  --image Ubuntu2204 \
  --admin-username $user \
  ../../../Users/pietr/.ssh/id_rsa.pub ../../../Users/pietr/.ssh/id_rsa \
  --public-ip-sku Standard
  
ip="$(az vm show \
  --resource-group $rg \
  --name $vm -d \
  --query [publicIps] \
  --output tsv)"
echo $ip

# Install the JDK.
## Log in to the virtual machine using an SSH tool.
ssh $user@$ip

## Install the JDK with apt
sudo apt-get update
sudo apt-get install -y default-jdk

## After installation is complete, run java -version to verify the Java environment. 
java -version

# See if password authentication is on
vi /etc/ssh/sshd_config

# Copy PC local PRIVATE SSH key to the Jenkins Server
$ scp ssh/id_rsa azureuser@51.136.28.189:/var/lib/jenkins/.ssh
id_rsa                  100% 1702    30.4KB/s   00:00

## Copy PC local PUBLIC SSH key to the Jenkins Agent, to allow connection from that to the agent
$ 
id_rsa.pub                                          100%  380     8.5KB/s   00:00    

## Must reduce permissions to avoid error:
chmod 400 ~/.ssh/id_rsa

# Ensure there's a known_hosts file, avoid
## No Known Hosts file was found at /var/lib/jenkins/.ssh/known_hosts
### Create Directory
sudo mkdir .ssh
### Let everyone read it (if not, only root can and scp will get permission denied)
sudo chmod o+w .ssh

### Copy the file
scp ssh/known_hosts azureuser@51.136.28.189:/var/lib/jenkins/.ssh

## Created directory in agent VM
/home/azureuser/jenkins-work
sudo mkdir /var/lib/jenkins
sudo mkdir /var/lib/jenkins/workspace
sudo mkdir /var/lib/jenkins/workspace/Spring
### Allow execute to all
sudo chmod o+x /var/lib/jenkins/workspace/Spring

# Install maven (not mentioned in article, but seemed to be missing)
    sudo apt update
    sudo apt install maven
    mvn -version

## SSH Credential
### Copy the private key file content EXACTLY into the private key field

# Add as Agent: see rest of article...