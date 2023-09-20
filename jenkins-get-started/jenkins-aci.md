# DATE: 18-09-2023
# SOURCE: https://learn.microsoft.com/en-us/azure/developer/jenkins/azure-container-instances-as-jenkins-build-agent
# RESULTS: WORKED!

# Configure Agent in Jenkins UI (see article)
## Agent Details
AGENT_NAME="ACIAgent"
JENKINS_SECRET="e276eba49d619dd72c340e46149d5f942385b45f710bdb0f002a70df95b21edc"

# Create ACI
  az login --tenant pietronromanolive.onmicrosoft.com
 ## Variables
  rg=jenkins-agent-rg
  aci=pnraci1
  jenkins_server_port="http://51.136.28.189:8080/"
 

az container create \
  --name $aci \
  --resource-group $rg \
  --ip-address Public --image jenkins/inbound-agent:latest \
  --os-type linux \
  --ports 80 \
  --command-line 'jenkins-agent -url ${jenkins_server_port} ${JENKINS_SECRET} ${AGENT_NAME}'


 # ARTICLE DIDN'T MENTION THIS BUT IS NEEDED TO LAUNCH THE AGENT
 ## Connect to the Jenkins Controller and rune
    curl -sO http://51.136.28.189:8080/jnlpJars/agent.jar
    sudo java -jar agent.jar -jnlpUrl http://51.136.28.189:8080/computer/ACIAgent/jenkins-agent.jnlp -secret b4edc4dc94acf3de4ba98afd7d38c80c9358fe7058f8fe3791c445baa4887051 -workDir "/home/jenkins/work"