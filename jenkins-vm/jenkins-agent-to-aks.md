# DATE: 01-10-2023
## Based loosely on Jenkins AKS Tutorial: deploy to AKS from agent using Azure CLI
## RESULT: SUCCESS! 

# REFERENCES:
- https://learn.microsoft.com/en-us/azure/developer/jenkins/deploy-from-github-to-aks

# logon
    agent_ip="13.95.168.237"
    user=jenkins
    ssh $user@$agent_ip

# Install Azure CLI on Agent
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    az --version


# Local: Create a Clust
## Login
    az login --tenant pietronromanolive.onmicrosoft.com

# Main Variables
    rg="jenkins-aks-rg"
    clu="pnrjenaks1"
    loc="westeurope"
    ns="app1-ns"

# Create RG and Cluster
    az group create -n $rg --location $loc
    az aks create -g $rg -n $clu --enable-managed-identity --node-count 1 --generate-ssh-keys

# Start/Stop the cluster
    az aks start -g $rg  -n $clu
    az aks stop -g $rg  -n $clu

## Get context
    az aks get-credentials --resource-group $rg --name $clu

## Step 1: Create a Namespace for App. 
    kubectl create namespace $ns


## Create a Service principal (note show does NOT retrieve password, have to reset)
    az ad sp list -o table
    az ad sp show --id  1ee946f5-ebe7-490e-837e-74ea36f556fa
    az ad sp credential reset --id  1ee946f5-ebe7-490e-837e-74ea36f556fa

## Created a new one
    az ad sp create-for-rbac 
    {
    "appId": "6f0d4fe8-82e6-44d6-9e46-51bf57e3def0",
    "displayName": "azure-cli-2023-10-01-14-49-23",
    "password": "Gw68Q~FNPxUJ_fFsfWALSbE3gzn-f0kzcyPDgbAA",
    "tenant": "599fd2f6-80be-4f0d-9b03-b3e74fdcf211"
    }
    az ad sp show --id "6f0d4fe8-82e6-44d6-9e46-51bf57e3def0"

## Important, need to create role assignment
### NOTE!!! DOUBLE SLASH // needed in Bash: https://stackoverflow.com/questions/69453392/az-tag-update-error-missingsubscription-the-request-did-not-have-a-subscripti
### OTHERWISE GET A MissingSubscription Error
az role assignment create \
    --assignee "2a3bb08c-9bb0-4a12-b6c3-a395801f9725" \
    --role "Contributor" \
    --scope "//subscriptions/26b58bc5-ccfe-48a9-b00e-51d89e19a4db/resourceGroups/jenkins-aks-rg"

    az role assignment create \
    --assignee "2a3bb08c-9bb0-4a12-b6c3-a395801f9725" \
    --role "Owner" \
    --scope "//subscriptions/26b58bc5-ccfe-48a9-b00e-51d89e19a4db"

    
# Agent: Install aks-cli
    sudo az aks install-cli

## Test some commands to put in the pipeline script
    az login --service-principal \
        -u "6f0d4fe8-82e6-44d6-9e46-51bf57e3def0" \
        -p "Gw68Q~FNPxUJ_fFsfWALSbE3gzn-f0kzcyPDgbAA" \
        --tenant "599fd2f6-80be-4f0d-9b03-b3e74fdcf211"

    az aks get-credentials --resource-group $rg --name $clu

    kubectl get pods -A
    kubectl run nginx --image=nginx -o yaml --dry-run=client > nginx-pod.yaml
    kubectl apply -f ./nginx-pod.yaml
    kubectl apply -f ./nodapp1-pod.yaml
    kubectl get pods

# Jenkins Controller VM
## Create Credential
 - Secret Text
 - sp_pwd
 - Gw68Q~FNPxUJ_fFsfWALSbE3gzn-f0kzcyPDgbAA

## Pipeline
### NOTE: DON'T USED pwd as credential name or variable, it gets masked 

pipeline {
    agent {label "jenkins-agent"}
    environment {
        rg = "jenkins-aks-rg"
        clu = "pnrjenaks1"
        sppwd = credentials("service-principal")
    }
    stages {
        stage('Deploy to AKS') {

            steps {
                sh '''
                    az login --service-principal \
                            -u "6f0d4fe8-82e6-44d6-9e46-51bf57e3def0" \
                            -p "${sppwd}" \
                            --tenant "599fd2f6-80be-4f0d-9b03-b3e74fdcf211"
                    
                    az aks get-credentials --resource-group "${rg}" --name "${clu}"
                    
                    kubectl get pods -A
                    kubectl run nginx --image=nginx -o yaml --dry-run=client > nginx-pod.yaml
                    kubectl apply -f ./nginx-pod.yaml
                    kubectl get pods > pods.txt
                '''
            }
        }
    }
}





