# DATE: 29-09-2023
## Tutorial CONTINUED FROM jenkins-vm-with-agents.md
## SOURCE: https://github.com/pietronromano/complete-prodcution-e2e-pipeline
## RESULT: SUCCESS! 
-  Sonar VM analyzed code fine
-

## REFERENCES:
- https://docs.dman.cloud/tutorial-documentation/install-sonarqube/


# Az Login
    az login --tenant pietronromanolive.onmicrosoft.com

# VM
## Variables
    rg=jenkins-vm-rg
    vm_sonar=sonar-vm
    user=sonar
    loc=westeurope

## Run az group create to create a resource group.
    az group create --name $rg --location westeurope

## Create Jenkins VM: need pwd initially to setup ssh between vms
    az vm create \
    --resource-group $rg \
    --name $vm_sonar \
    --image Ubuntu2204 \
    --admin-username $user \
    --admin-password Axml-xsl0123 \
    --public-ip-sku Standard
  
## Enable auto-shutdown
    az vm auto-shutdown -g $rg -n $vm_sonar --time 2300 --email "pietronromano@live.com" 


## Get VM IP
  ip="$(az vm show \
    --resource-group $rg \
    --name $vm_sonar -d \
    --query [publicIps] \
    --output tsv)"
  echo $ip
  23.97.142.202

## 9000
    az vm open-port \
    --resource-group $rg \
    --name $vm_sonar \
    --port 9000 --priority 1010

## Log in to the virtual machine using an SSH tool.
    ssh $user@$ip


## Update Package Repository and Upgrade Packages
    sudo apt update
    sudo apt upgrade

# PostgreSQL
## Add PostgresSQL repository
    sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null

## Install PostgreSQL
    sudo apt update
    sudo apt-get -y install postgresql postgresql-contrib
    sudo systemctl enable postgresql

## Create Database for Sonarqube
### Set password for postgres user
    sudo passwd postgres Axml-xsl0123

### Change to the postgres user
    su - postgres

## Create database user postgres
    createuser sonar

### Set password and grant privileges
    createuser sonar
    psql 
    ALTER USER sonar WITH ENCRYPTED password 'sonar';
    CREATE DATABASE sonarqube OWNER sonar;
    grant all privileges on DATABASE sonarqube to sonar;
    \q
    exit

# Adoptium Java 17
# Switch to root user
    sudo sonar
    sudo -i

## Add adoptium repository
    wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
    echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list

# Install Java 17
## Update repository and install Java
    apt update
    apt install temurin-17-jdk
    update-alternatives --config java
    /usr/bin/java --version
    exit 

# Linux Kernel Tuning¶
## Increase Limits¶
    sudo vim /etc/security/limits.conf

### Paste the below values at the bottom of the file
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096

## Increase Mapped Memory Regions
    sudo vim /etc/sysctl.conf

## Paste the below values at the bottom of the file
vm.max_map_count = 262144

# Reboot System
    sudo reboot

# Sonarqube
## Download and Extract
    sonar_zip=sonarqube-10.2.1.78527
    sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
    sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/$sonar_zip.zip
    sudo apt install unzip
    sudo unzip $sonar_zip.zip -d /opt

    sudo mv /opt/$sonar_zip /opt/sonarqube

# Create user and set permissions
    sudo groupadd sonar
    sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
    sudo chown sonar:sonar /opt/sonarqube -R

## Update Sonarqube properties with DB credentials
    sudo vim /opt/sonarqube/conf/sonar.properties



## Find and replace the below values, you might need to add the sonar.jdbc.url
### CAREFUL!!! MUST USE "sonar" as password, not the linux user pwd, otherwise get "+HeapDumpOnOutOfMemoryError"
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube

# Create service for Sonarqube
    sudo vim /etc/systemd/system/sonar.service

## Paste the below into the file
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target

## Start Sonarqube and Enable service
    sudo systemctl start sonar
    sudo systemctl enable sonar
    sudo systemctl status sonar

## Watch log files and monitor for startup
    sudo tail -f /opt/sonarqube/logs/sonar.log

# Access the Sonarqube UI
    http://23.97.142.202:9000

## User: admin, pwd: admin -> Axml-xsl0123
### Create API: SEE https://youtu.be/q4g7KJdFSn0?t=3681
- Top Right Adminsitrator -> My Account -> Security -> Generate Token
   - name: jenkins
   - Global Analysis
   - No Expiration
   - Copy: sqa_eed74e05eadd6a5445fb361cd303c6724caac35f

# Jenkins Controller
## Create Credential -> Secret Text
- Paste in secret
  - id: jenkins-sonarqube-token

## Add plugins
- Sonarqube Scanner
- Sonar Quality Gates
- Quality Gates

### Configure System
- Sonarqube Location: System
  - http://23.97.142.202:9000
  - Use Environment Variables
  - name: sonarqube-scanner
  - token: jenkins-sonarqube-token
    - Apply
- Scanner: Tools
  - SonarQube Scanner Installations
    - sonarqube-scanner-latest
    - Apply

# Update Pipeline
        stage("Sonarqube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

# Quality Gate  / Webhook
    https://youtu.be/q4g7KJdFSn0?t=5104

## Sonarqube Webhook: (ensure it has / on the end)
 - Administration -> Configuration -> Webhooks
   - URL of: http://52.166.179.138:8080/sonarqube-webhook/

## Add Quality Gate to Jenkinsfile
      stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }

        }