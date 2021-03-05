# Build and deploy Azure DevOps Pipeline Agent on AKS
<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-2-orange.svg?style=flat-square)](#contributors-)
<!-- ALL-CONTRIBUTORS-BADGE:END -->

### Description:
This project details the steps for customizing the Azure DevOps Pipeline Agent and deploying the same on Azure Kubernetes Service.  AKS is a managed kubernetes service on Azure.

### Prerequisites:
1. A Linux VM with the following tools installed.
   - Azure CLI
   - Kubernetes CLI (kubectl)
   - Docker engine
   - Helm CLI v3 (Kubernetes Package Manager)

2. A Microsoft Azure Account and access to the Azure Portal

3. An Azure DevOps Services account and access to the Azure DevOps Services Portal

4. Access to a Azure Container Registry (ACR) instance

5. Access to an Azure Kubernetes Cluster (AKS) instance.

### A] Create PAT token and Agent Pool in Azure DevOps Services

1. Login to Azure DevOps Services portal with your credentials

2. Create a *PAT* token

   Click on your profile name (top right corner) and then click on **Security**.  Click on **+ New Token**. See screenshot below.

   ![alt tag](./images/A-01.PNG)

   In the *Create a personal access token* window, specify a name for the PAT Token, set the **Expiration** field to **Custom defined** and select an expiry date.  Set the **Scopes** field to **Full access**.  Then click on **Create**.  See screenshot below.

   ![alt tag](./images/A-02.PNG)

   Copy and save the PAT token.  The token won't be accessible once the window is closed.

   ![alt tag](./images/A-03.PNG)

2. Create an *Agent Pool*

   Open up the Azure DevOps organization tab and then click on **Organization settings** (lower left corner).  See screenshot below.   

   ![alt tag](./images/A-04.PNG)

   Then click on **Agent pools** as shown in the screenshot below.

   ![alt tag](./images/A-05.PNG)

   Click on **Add pool**, specify a name for the agent pool and check the box for **Grant access permission to all pipelines**.  Click on **Create**.  See screenshots below.

   ![alt tag](./images/A-06.PNG)

   ![alt tag](./images/A-07.PNG)

### B] Build Azure DevOps Pipeline Agent and push it to Azure Container Registry (ACR)

Login to the Linux VM (via SSH) containing the CLI tools (outlined in the *Prerequisites* section) before proceeding with the next steps.

1. Download (Copy) the Azure DevOps Pipeline Agent

   Refer to [Azure DevOps docs](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops) and follow the steps to copy the `Dockerfile` and `start.sh` scripts to a local VM with **docker** engine installed on it.

   The files are also provided in the `./dockeragent` directory. You will need to clone/fork this repository in order to use the provided scripts to build the Azure Pipeline Agent.
   The provided `Dockerfile` installs **latest** versions of the following tools.
   - [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest)
   - [Docker CE Client](https://docs.docker.com/install/)
   - [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
   - [Helm](https://helm.sh/)
   - [Aqua Trivy](https://github.com/aquasecurity/trivy) Container image scanner

2. Build the Azure DevOps Pipeline agent

   ```bash
   # Change directory to './dockeragent'
   $ cd ./dockeragent
   #
   # Build the Azure DevOps Pipeline Agent
   $ docker build -t azdevopsagent:latest .
   #
   # List the docker images
   $ docker images
   #
   ```

3. Push the Azure DevOps Pipeline Agent container image to ACR

   ```bash
   # NOTE: Substitute the correct value for ACR ('<acrName>') in the commands below
   #
   # Login to ACR with your credentials.
   $ az acr login --name <acrName>
   #
   # Tag the Azure DevOps Pipeline image 
   $ docker tag azdevopsagent:latest <acrName>.azurecr.io/azdevopsagent:v1
   #
   # List the docker images
   $ docker images
   #
   # Push the image to ACR
   $ docker push <acrName>.azurecr.io/azdevopsagent:v1
   #
   # List the images in the ACR repository
   $ az acr repository list --name <acrName> -o table
   #
   # List the tags in the 'azdevopsagent' repository
   $ az acr repository show-tags --name <acrName> --repository azdevopsagent -o table
   #
   ```

### C] Test the Azure DevOps Pipeline Agent on a local VM

1. Login to the Linux VM via a terminal session.

2. Run the Azure DevOps Pipeline Agent
   
   ```bash
   #
   # Provide correct values for the following parameters
   #
   # AZP_URL = Azure DevOps URL eg., https://dev.azure.com/org
   # AZP_TOKEN = Azure DevOps PAT Token
   # AZP_AGENT_NAME = Any meaningful name for the container agent
   # AZP_POOL = Name of the agent pool registered in Azure DevOps Services.  Default value is 'Default' pool
   #
   # Use docker bind mount for running docker builds within the docker agent container! Add the '-v' option as below.
   # -v /var/run/docker.sock:/var/run/docker.sock
   #
   $ docker run -e AZP_URL="<Azure DevOps URL>" -e AZP_TOKEN="<Azure DevOps PAT Token>" -e AZP_AGENT_NAME="Az-DevOps-Agent" -e AZP_POOL="<Azure DevOps Agent Pool name>" azdevopsagent:latest
   #
   ```

3. Verify the agent has registered with the correct pool in Azure DevOps Services

   Login to Azure DevOps Services Portal with your credentials.  Click on **Organization Settings** and then click on **Agent pools**.  See screenshot below.

   ![alt tag](./images/C-01.png)

   From the agent pool list, select the pool the agent is connected to.

   ![alt tag](./images/C-02.png)

   Click on the **Agents** tab and make sure the agent is online.  See screenshot below. 

   ![alt tag](./images/C-03.png)

### D] Deploy Azure DevOps Pipeline Agent on AKS
In this step, the Kubernetes package manager *Helm* will be used to deploy the Azure DevOps Pipeline agent on AKS.

1. Update the following values in *Helm* chart file `./devops-agent/values.yaml`

   | Parameter Name | Description |
   |----------------|-------------|
   | **replicaCount** | No. of agent instances eg., 1..N |
   | **repository** | ACR Name eg., **acrname**.azurecr.io/azdevopsagent |
   | **tag** | Azure DevOps Pipeline Agent Container image tag name |
   | **azpUrl** | Azure DevOps URL eg., https://dev.azure.com/org-name |
   | **azpToken** | Azure DevOps PAT Token (Created in Step A.2) |
   | **azpPool** | Name of the agent pool registered in Azure DevOps Services.  Defaults to 'Default' pool |

2. Deploy the Azure DevOps Pipeline Agent on AKS

   Login to the Linux VM via a terminal session (in case you haven't already).  You must have `kubectl` and `helm` configured and connected to the AKS cluster.

   ```bash
   #
   # Create a new namespace 'az-devops'
   $ kubectl create namespace az-devops
   #
   # Deploy the Azure Pipeline Agent.
   # If you are using Helm v3.x for deploying the pipeline agent then use the command below.
   $ helm install --namespace az-devops az-devops ./devops-agent/
   #
   # If you are using Helm v2.x for deploying the pipeline agent then use the command below.
   # helm install ./devops-agent/ --namespace az-devops -n az-devops
   #
   # List the helm application deployments (releases) in 'az-devops' namespace
   $ helm list -n az-devops
   #
   # List the Pods in 'az-devops' namespace
   $ kubectl get pods -n az-devops
   #
   # View the agent container logs.  Substitute the Pod name in the command below.
   $ kubectl logs <Pod-Name> -n az-devops
   1. Determining matching Azure Pipelines agent...
   2. Downloading and installing Azure Pipelines agent...
   3. Configuring Azure Pipelines agent...

   >> End User License Agreements:

   Building sources from a TFVC repository requires accepting the Team Explorer Everywhere End User License Agreement. This step is not required for building sources from Git repositories.

   A copy of the Team Explorer Everywhere license agreement can be found at:
     /azp/agent/externals/tee/license.html


   >> Connect:

   Connecting to server ...

   >> Register Agent:

   Scanning for tool capabilities.
   Connecting to the server.
   Successfully replaced the agent
   Testing agent connection.
   2019-08-18 01:29:29Z: Settings Saved.
   4. Running Azure Pipelines agent...
   Starting Agent listener interactively
   Started listener process
   Started running service
   Scanning for tool capabilities.
   Connecting to the server.
   2019-08-18 01:29:32Z: Listening for Jobs
   #

   ```
3. Verify the agent has registered with the correct pool in Azure DevOps Services

   Refer to Step C.2 (above).
   
### Summary
To recap, this project detailed the steps for customizing the *Azure DevOps Pipeline Agent* (use a custom Dockerfile), building and testing the agent locally on a Linux VM and then deploying the agent container on AKS.
By running multiple Azure DevOps Pipeline agent instances (containers) on AKS, multiple build jobs can be run concurrently in parallel and an organization's application build infrastructure can be elastically scaled. 

## Contributors ✨

Thanks to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)) for their contributions:

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tr>
    <td align="center"><a href="https://github.com/naustin"><img src="https://avatars0.githubusercontent.com/u/1503072?v=4" width="100px;" alt=""/><br /><sub><b>Noel Austin</b></sub></a><br /><a href="https://github.com/ganrad/Az-DevOps-Agent-On-AKS/commits?author=naustin" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/Krupakarreddin"><img src="https://avatars0.githubusercontent.com/u/15169411?v=4" width="100px;" alt=""/><br /><sub><b>Krupakarreddin</b></sub></a><br /><a href="https://github.com/ganrad/Az-DevOps-Agent-On-AKS/issues?q=author%3AKrupakarreddin" title="Bug reports">🐛</a></td>
  </tr>
</table>

<!-- markdownlint-enable -->
<!-- prettier-ignore-end -->
<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
