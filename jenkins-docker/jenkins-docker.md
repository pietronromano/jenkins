# DATE: 19-09-2023
# MY IDEA: Create Ubuntu VM in Azure, then install Docker

# Create a virtual machine
## Create a test directory called jenkins-get-started.
cd jenkins-docker

# CLI - Create VM
## Login
  az login --tenant pietronromanolive.onmicrosoft.com
  az account set --subscription "26b58bc5-ccfe-48a9-b00e-51d89e19a4db"

## Variables
  rg=jenkins-docker-rg
  vm=jenkins-docker-vm
  loc=westeurope
  user=azureuser

## Run az group create to create a resource group.
  az group create --name $rg --location westeurope

## List VM images with Ubuntu
  az vm image list --query \
  "[].{Alias:urnAlias, SKU: sku, Version: version} | [? contains(Alias,'Ubuntu')]" \
  -o table

## Run az vm create to create a virtual machine. 
  az vm create \
  --resource-group $rg \
  --name $vm \
  --image Ubuntu2204 \
  --admin-username $user \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --custom-data cloud-init-docker.txt


## Enable auto-shutdown
  az vm auto-shutdown -g $rg -n $vm --time 1900 --email "pietronromano@live.com" 

### Run az vm list to verify the creation (and state) of the new virtual machine.
  az vm list -d -o table --query "[?name=='${vm}']"

### As Jenkins runs on port 8080, run az vm open to open port 8080 on the new virtual machine.


# Configure Jenkins
## Run az vm show to get the public IP address for the sample virtual machine.
### MINE: Save the ip to a variable, tsv removes the quotes
ip="$(az vm show \
  --resource-group $rg \
  --name $vm -d \
  --query [publicIps] \
  --output tsv)"
echo $ip

## Login with SSH
ssh $user@$ip
exit

# Install Docker
## SOURCE: https://docs.docker.com/engine/install/ubuntu/

# Deallocate (Stop is still billed) / Delete the VM / RG
  az vm deallocate --resource-group $rg  --name $vm 
  az vm start --resource-group $rg  --name $vm 
  
  az vm delete --resource-group $rg  --name $vm 
  az group delete -g $rg