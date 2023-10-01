# DATE: 13-09-2023
# SOURCE: https://learn.microsoft.com/en-us/azure/developer/jenkins/configure-on-linux-vm
# RESULT: SUCESS! Managed to create pipelines

# Create a virtual machine
## Create a test directory called jenkins-get-started.
mkdir jenkins-get-started
cd jenkins-get-started

## Create a file named cloud-init-jenkins.txt.

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

## Run az vm create to create a virtual machine
## Run Jenkins directly in the VM
### cloud-init-jenkins: included jenkins and docker
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
  az vm list -d -o table --query "[?name=='${vm}']"

### As Jenkins runs on port 8080, run az vm open to open port 8080 on the new virtual machine.
  az vm open-port \
  --resource-group $rg \
  --name $vm  \
  --port 8080 --priority 1010

  # Jenkins agents connect to the controller via port 50000, ensure that port is allowed inbound to the Jenkins Controller.
  az vm open-port \
  --resource-group $rg \
  --name $vm  \
  --port 50000 --priority 1050

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

## Verify Docker
  sudo docker
### If omitted from init, need to add Jenkins to docker group for plugins, then restart Jenkins
  - sudo usermod -a -G docker jenkins

## Verify that Jenkins is running by getting the status of the Jenkins service.
  sudo service jenkins status

## Get the admin pwd
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword

## Using the IP address, open the following URL in a browser: 
  echo "http://$ip:8080"
  Enter the password you retrieved earlier and select Continue.

  User: pietronromano, Travies1
  http://20.86.163.168:8080/

# My Test with NodeApps: Worked OK
## Install nodejs
  sudo apt-get update
  sudo apt-get install nodejs
  sudo apt install npm

## In Jenkins UI, install nodejs, then plugin, restart Jenkins
## Create nodejs job
  https://github.com/wardviaene/docker-demo.git

### Output
  cd /var/lib/jenkins/workspace/nodeapp

# Simple Maven App: 
# RESULT: SUCCESS!
## Create a pipeline with SCM
## JenkinsFile: jenkins/Jenkinsfile
## https://github.com/jenkins-docs/simple-java-maven-app

## Test the app
  cd /var/lib/jenkins/workspace/SimpleJavaMaven/target$
  java -jar my-app-1.0-SNAPSHOT.jar
  Hello World!


# Deallocate (Stop is still billed) / Delete the VM / RG
  az vm deallocate --resource-group $rg  --name $vm 
  az vm start --resource-group $rg  --name $vm 
  
  az vm delete --resource-group $rg  --name $vm 
  az group delete -g $rg
