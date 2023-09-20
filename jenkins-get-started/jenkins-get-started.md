# DATE: 13-09-2023
# SOURCE: https://learn.microsoft.com/en-us/azure/developer/jenkins/configure-on-linux-vm

# Create a virtual machine
## Create a test directory called jenkins-get-started.
mkdir jenkins-get-started
cd jenkins-get-started

## Create a file named cloud-init-jenkins.txt.
### Paste the following code into the new file:
#cloud-config
package_upgrade: true
runcmd:
  - sudo apt install openjdk-11-jre -y
  - curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
  -  echo 'deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/' | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
  - sudo apt-get update && sudo apt-get install jenkins -y
  - sudo service jenkins restart

# CLI - Create VM
## Login
  az version
  az upgrade
  az login --tenant pietronromanolive.onmicrosoft.com
  az account set --subscription "26b58bc5-ccfe-48a9-b00e-51d89e19a4db"

## Variables
  rg=jenkins-get-started-rg
  vm=jenkins-get-started-vm
  loc=westeurope

## Run az group create to create a resource group.
  az group create --name $rg --location westeurope

## List VM images with Ubuntu
  az vm image list --query \
  "[].{Alias:urnAlias, SKU: sku, Version: version} | [? contains(Alias,'Ubuntu')]" \
  -o table

## Run az vm create to create a virtual machine. 
### NOTE: Article uses UbuntuLTS (which also isn't compatible with latest versions of node), but CLI advises Ubuntu2204
  az vm create \
  --resource-group $rg \
  --name $vm \
  --image Ubuntu2204 \
  --admin-username "azureuser" \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --custom-data cloud-init-jenkins.txt

## Enable auto-shutdown
  az vm auto-shutdown -g $rg -n $vm --time 1900 --email "pietronromano@live.com" 

### Run az vm list to verify the creation (and state) of the new virtual machine.
  az vm list -d -o table --query "[?name=='jenkins-get-started-vm']"

### As Jenkins runs on port 8080, run az vm open to open port 8080 on the new virtual machine.
  az vm open-port \
  --resource-group $rg \
  --name $vm  \
  --port 8080 --priority 1010

  # Jenkins agents connect to the controller via port 5000, ensure that port is allowed inbound to the Jenkins Controller.
  az vm open-port \
  --resource-group $rg \
  --name $vm  \
  --port 5000 --priority 1020

# Configure Jenkins
## Run az vm show to get the public IP address for the sample virtual machine.
### MINE: Save the ip to a variable, tsv removes the quotes
ip="$(az vm show \
  --resource-group $rg \
  --name $vm -d \
  --query [publicIps] \
  --output tsv)"
echo $ip

## Login
ssh azureuser@$ip

# Upon successful connection, the Cloud Shell prompt includes the user name and virtual machine name: azureuser@jenkins-get-started-vm.

## Verify that Jenkins is running by getting the status of the Jenkins service.
service jenkins status

## Get the admin pwd
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

## Using the IP address, open the following URL in a browser: 
http://51.136.28.189:8080/
Enter the password you retrieved earlier and select Continue.

# My Test with NodeApps: Worked OK
## Install nodejs
  sudo apt-get update
  sudo apt-get install nodejs
  sudo apt install npm

## In Jenkins UI, install nodejs, then plugin, restart Jenkins
## Create nodejs job
  https://github.com/wardviaene/docker-demo.git

## Output
  cd /var/lib/jenkins/workspace/nodeapp

## My Result with Ubuntu2004: Gradle Build failed
* What went wrong:
A problem occurred configuring root project 'spring-boot'.
> Could not resolve all files for configuration ':classpath'.
   > Could not resolve org.springframework.boot:spring-boot-gradle-plugin:3.1.0.
     Required by:

# Deallocate (Stop is still billed) / Delete the VM / RG
  az vm deallocate --resource-group $rg  --name $vm 
  az vm start --resource-group $rg  --name $vm 
  
  az vm delete --resource-group $rg  --name $vm 
  az group delete -g $rg
