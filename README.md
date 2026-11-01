# CI/CD Project: Deploy a Microservices Voting Application on Azure Kubernetes Service Using Azure DevOps and Argo CD

![Azure DevOps](https://img.shields.io/badge/Azure%20DevOps-CI%2FCD-blue)
![Azure Kubernetes Service](https://img.shields.io/badge/Azure-AKS-0078D4)
![Azure Container Registry](https://img.shields.io/badge/Azure-ACR-0078D4)
![Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-orange)
![Kubernetes](https://img.shields.io/badge/Container%20Orchestration-Kubernetes-326CE5)

## Project Overview

This project demonstrates how to implement an automated CI/CD and GitOps workflow to deploy a microservices-based voting application to **Azure Kubernetes Service (AKS)**.

The solution uses **Azure DevOps Pipelines** to build and push Docker images to **Azure Container Registry (ACR)**. A Bash script then updates the Kubernetes deployment manifests stored in Azure Repos. **Argo CD** continuously monitors the repository and automatically synchronizes changes with the AKS cluster.

The project demonstrates:

- Building and pushing Docker images using Azure DevOps.
- Configuring a self-hosted Azure DevOps agent on an Azure Linux VM.
- Storing container images in Azure Container Registry.
- Deploying microservices to Azure Kubernetes Service.
- Implementing GitOps with Argo CD.
- Automating Kubernetes manifest updates through Bash scripting.
- Configuring AKS to pull private images from ACR.
- Exposing the application using Kubernetes NodePort services.

## Architecture

### 1. Application Architecture

![alt text](<architecture/application architecture.png>)

The application consists of five main components:

| Component | Technology | Responsibility |
|---|---|---|
| Vote | Python / Flask | Provides the interface for users to submit votes. |
| Redis | Redis | Temporarily stores incoming votes. |
| Worker | .NET | Processes votes from Redis and writes the results to the database. |
| Database | PostgreSQL | Stores the processed voting results. |
| Result | Node.js | Displays the voting results to users. |

### Application workflow

1. A user accesses the Vote application and submits a vote.
2. The Vote service sends the vote to Redis.
3. Redis temporarily stores the vote.
4. The Worker retrieves votes from Redis and processes them.
5. The Worker writes the processed results to PostgreSQL.
6. The Result application retrieves and displays the voting results.

Each component runs independently as a containerized microservice.

### 2. CI/CD and GitOps Architecture

![alt text](<architecture/Azure DevOps Vote App Pipeline Diagram..png>)

The deployment process is divided into two major stages:

**Continuous Integration (CI)**

1. The developer pushes application code to the source repository.
2. The application source code is available in Azure Repos.
3. Azure DevOps pipelines detect changes in the relevant microservice directories.
4. The pipelines execute on a self-hosted agent hosted on an Azure Linux VM.
5. Docker images are built for the Vote, Result, and Worker services.
6. The images are pushed to Azure Container Registry.
7. A Bash script updates the corresponding Kubernetes deployment manifest with the newly built image tag.
8. The updated manifest is committed and pushed to Azure Repos.

**Continuous Delivery (CD)**

1. Argo CD continuously monitors the Kubernetes manifest directory in Azure Repos.
2. Argo CD detects changes to the deployment manifests.
3. Argo CD automatically synchronizes the updated manifests with AKS.
4. AKS pulls the required container images from ACR.
5. Kubernetes creates or updates the application pods.
6. Users access the Vote and Result applications through their respective NodePort services.

## Technologies and Services Used

- **Source code:** GitHub – Docker Example Voting App
- **Source and deployment repository:** Azure Repos
- **CI/CD:** Azure DevOps Pipelines
- **Build agent:** Self-hosted Azure DevOps agent on Ubuntu Linux VM
- **Containerization:** Docker
- **Container registry:** Azure Container Registry (ACR)
- **Container orchestration:** Azure Kubernetes Service (AKS)
- **GitOps:** Argo CD
- **Scripting:** Bash
- **Application components:** Python, Node.js, .NET, Redis and PostgreSQL

## Project Implementation

# Stage One: Continuous Integration

## Step 1: Get the Application Source Code

The project uses the public Docker Example Voting App repository:

[https://github.com/dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)

Clone the repository:

```bash
git clone https://github.com/dockersamples/example-voting-app.git

cd example-voting-app
```

The repository contains the application source code and Dockerfiles for the different microservices.

The application is divided into separate directories, including:

```text
example-voting-app/
├── vote/
├── result/
├── worker/
├── docker-compose.yml
└── ...
```

Each service has its own Dockerfile, which is used by the CI pipelines to build its container image.

## Step 2: Create an Azure DevOps Project and Import the Repository

1. Sign in to [Azure DevOps](https://dev.azure.com).
2. Create a new project.
3. Navigate to **Repos → Files**.
4. Select the option to import a repository.
5. Enter the GitHub repository URL:

   ```text
   https://github.com/dockersamples/example-voting-app.git
   ```

6. Import the repository into Azure Repos.

For this implementation, the Azure DevOps project and repository are:

```text
Project: Azure-project
Repository: Azure-project
Organization: Ifedayo001
```

Azure Repos serves as the central repository for the application source code and Kubernetes deployment manifests used in this project.

## Step 3: Create Azure Container Registry

![alt text](<screenshots/azure container registry created.png>)

Azure Container Registry provides a private location to store the Docker images built by the pipelines.

1. Open the Azure Portal.
2. Search for **Container Registries**.
3. Select **Create**.
4. Choose the subscription and resource group.
5. Provide a unique registry name.
6. Select the required pricing tier.
7. Review and create the registry.

The ACR configuration used in this project:

```text
ACR Name: ifedayoazurecicd
Login Server: ifedayoazurecicd.azurecr.io
```

The registry stores the images generated by the three microservice pipelines.

## Step 4: Set Up a Self-Hosted Azure DevOps Agent

![alt text](<screenshots/azureagent vm created to run the CI pipelines.png>)

A self-hosted agent is required to execute the Azure DevOps pipeline jobs.

Instead of using Microsoft-hosted agents, I created an Ubuntu Linux virtual machine in Azure and configured it as a self-hosted agent.

### 4.1 Create the Agent Pool

![alt text](<screenshots/azureagent successfully connected to run CI pipelines.png>)

1. Navigate to **Azure DevOps → Project Settings**.
2. Select **Agent Pools**.
3. Create a new self-hosted agent pool.
4. Name the pool:

   ```text
   azureagent
   ```

5. Open the newly created pool and select **New Agent**.
6. Select Linux and follow the instructions provided by Azure DevOps.

### 4.2 Configure the Agent on the Ubuntu VM

![alt text](<screenshots/azureagent configured and listening for jobs.png>)

Download and extract the agent package using the instructions provided by Azure DevOps.

For example:

```bash
mkdir myagent
cd myagent

# Download the agent package using the URL
# provided by Azure DevOps.

tar zxvf vsts-agent-linux-x64-<version>.tar.gz
```

Configure the agent:

```bash
./config.sh
```

During configuration:

- Enter the Azure DevOps organization URL.
- Select the authentication method requested by the setup wizard.
- Provide the required authentication credentials.
- Select the `azureagent` pool.
- Complete the remaining configuration prompts.

Start the agent:

```bash
./run.sh
```

Once the agent is running, its status should show as **Online** in Azure DevOps.

> The agent VM must have Docker installed and the required permissions to build and push container images.

## Step 5: Create Azure DevOps CI Pipelines for the Microservices

Three separate pipelines were created to build and push the application images:

1. Vote pipeline
2. Result pipeline
3. Worker pipeline

Each pipeline is triggered by changes to its corresponding application directory.

### 5.1 Vote Pipeline

The Vote pipeline builds and pushes the Vote application image.

The pipeline uses the following configuration:

```yaml
trigger:
  paths:
    include:
      - vote/*

resources:
- repo: self

variables:
  dockerRegistryServiceConnection: '<ACR_SERVICE_CONNECTION_ID>'
  imageRepository: 'votingapp'
  containerRegistry: 'ifedayoazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'azureagent'

stages:
- stage: Build
  displayName: Build the image
  jobs:
  - job: Build
    steps:
    - task: Docker@2
      displayName: Build the image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'vote/Dockerfile'
        tags: '$(tag)'

- stage: Push
  displayName: Push the image
  jobs:
  - job: Push
    steps:
    - task: Docker@2
      displayName: Push the image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'push'
        tags: '$(tag)'

- stage: Update
  displayName: Update Kubernetes manifest
  jobs:
  - job: Update
    steps:
    - task: ShellScript@2
      inputs:
        scriptPath: 'scripts/updatek8sManifests.sh'
        args: 'vote $(imageRepository) $(tag)'
```

Replace the service connection placeholder with the actual Docker Registry service connection ID configured in Azure DevOps.

### 5.2 Result Pipeline

The Result pipeline follows the same process but targets the Result microservice.

Its main configuration differences are:

```yaml
trigger:
  paths:
    include:
      - result/*

variables:
  imageRepository: 'resultapp'
  containerRegistry: 'ifedayoazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'
```

The Docker build and push tasks use:

```yaml
Dockerfile: 'result/Dockerfile'
```

The manifest update task passes:

```yaml
args: 'result $(imageRepository) $(tag)'
```

### 5.3 Worker Pipeline

The Worker pipeline follows the same structure.

It monitors changes to the Worker directory, builds the Worker Docker image, pushes it to ACR, and updates its Kubernetes deployment manifest.

The image repository name and Dockerfile path should match the actual Worker image configuration in ACR and the repository.

### 5.4 Pipeline Execution

Each pipeline contains three stages:

| Stage | Description |
|---|---|
| Build | Builds the Docker image from the microservice Dockerfile. |
| Push | Pushes the newly built image to Azure Container Registry. |
| Update | Runs the Bash script to update the corresponding Kubernetes manifest in Azure Repos. |

The pipeline uses `$(Build.BuildId)` as the image tag, ensuring that each build has a unique version.

After successful execution, the images can be viewed under **Azure Portal → Container Registry → Repositories**.

# Stage Two: Continuous Delivery

## Step 6: Create an Azure Kubernetes Service (AKS) Cluster

![alt text](<screenshots/Kubernetes cluster created.png>)

Azure Kubernetes Service provides the managed Kubernetes environment where the application will run.

1. Open the Azure Portal.
2. Search for **Kubernetes Services**.
3. Select **Create**.
4. Choose the subscription and resource group.
5. Provide a cluster name.
6. Select the required region and node configuration.
7. Review and create the cluster.

The AKS cluster created for this project:

```text
Cluster Name: azuredevops
Resource Group: azurecicd
```

Wait for the deployment to complete before proceeding.

## Step 7: Configure Azure CLI and Connect to AKS

![alt text](<screenshots/Loging to AKS and setting up ArgoCD.png>)

To manage the Kubernetes cluster, Azure CLI and kubectl are required.

Install Azure CLI on the Ubuntu workstation:

```bash
sudo apt-get update

curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Verify the installation:

```bash
az --version
```

Log in to Azure:

```bash
az login --use-device-code
```

Install kubectl:

```bash
sudo az aks install-cli
```

Connect to the AKS cluster:

```bash
az aks get-credentials \
  --resource-group azurecicd \
  --name azuredevops \
  --overwrite-existing
```

Verify the connection:

```bash
kubectl get nodes
```

The cluster nodes should appear with a `Ready` status.

## Step 8: Install Argo CD on AKS

![alt text](<screenshots/configuring ArgoCD and setting the service type to NodePort for web access.png>)

Argo CD is used to implement GitOps by continuously monitoring the Kubernetes manifests stored in Azure Repos.

Create a namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the Argo CD components to become ready:

```bash
kubectl wait \
  --for=condition=Ready pods \
  --all \
  -n argocd \
  --timeout=600s
```

Verify the installation:

```bash
kubectl get pods -n argocd
```

## Step 9: Expose Argo CD Using NodePort

![alt text](<screenshots/configuring ArgoCD and setting the service type to NodePort for web access 2.png>)

To access the Argo CD web interface externally, the Argo CD server service was changed to NodePort.

Run:

```bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
```

Retrieve the assigned NodePort:

```bash
kubectl get svc argocd-server -n argocd
```

Retrieve the initial administrator password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

The default username is:

```text
admin
```

### Configure the Azure Network Rule

![alt text](<screenshots/setting up vmss to allow inbound traffic to allow access.png>)

Since the Argo CD service is exposed through NodePort, the required port must be accessible through the Azure network configuration.

1. Open the Azure Portal.
2. Navigate to the AKS node pool's Virtual Machine Scale Set.
3. Open its networking settings.
4. Add an inbound security rule for the assigned Argo CD NodePort.
5. Restrict the source IP range where possible.

Access Argo CD using:

```text
http://<NODE_PUBLIC_IP>:<ARGOCD_NODEPORT>
```
![alt text](<screenshots/successfull login to ArgoCD.png>)

Use the administrator credentials to sign in.

> For a production environment, use HTTPS, restrict network access, and configure appropriate authentication instead of exposing the Argo CD interface publicly.

## Step 10: Connect Argo CD to Azure Repos

![alt text](<screenshots/Azuredevops Repo successfully connected to ArgoCD.png>)

Argo CD needs access to the repository containing the Kubernetes deployment manifests.

1. Log in to the Argo CD web interface.
2. Navigate to **Settings → Repositories**.
3. Select **Connect Repo**.
4. Choose HTTPS as the connection method.
5. Enter the Azure Repos repository URL.
6. Provide the required repository credentials.
7. Test the connection.

Repository URL:

```text
https://dev.azure.com/Ifedayo001/Azure-project/_git/Azure-project
```

For private repositories, provide an appropriately scoped credential with read access.

Store credentials securely. Do not commit personal access tokens or passwords to the repository.

## Step 11: Create an Argo CD Application

![alt text](<screenshots/application created in ArgoCD.png>)

After connecting Argo CD to Azure Repos, create an application that monitors the Kubernetes manifest directory.

In the Argo CD interface:

1. Select **New Application**.
2. Provide an application name.
3. Select the `default` project.
4. Select the connected Azure Repos repository.
5. Set the target revision to the deployment branch.
6. Set the path to:

   ```text
   k8s-specifications
   ```

7. Set the destination cluster to the local AKS cluster.
8. Set the destination namespace to:

   ```text
   default
   ```

9. Select the automatic sync policy.
10. Create the application.

Argo CD will now monitor the specified repository path and synchronize changes with AKS.

## Step 12: Create an ACR Image Pull Secret

![alt text](<screenshots/ImagePullBackOff error due to ACR and AKS not yet authenticated.png>)

![alt text](<screenshots/secret created to authenticate ACR and AKS.png>)

One of the challenges encountered during deployment was `ImagePullBackOff`.

The issue occurred because AKS could not authenticate to the private Azure Container Registry when attempting to pull the application images.

To resolve this, a Kubernetes image pull secret was created.

Create the secret using the appropriate ACR credentials:

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=ifedayoazurecicd.azurecr.io \
  --docker-username='<ACR_USERNAME>' \
  --docker-password='<ACR_PASSWORD>' \
  --docker-email='<EMAIL_ADDRESS>' \
  --namespace=default
```

Use secure credentials and avoid storing them in source control.

Verify the secret:

```bash
kubectl get secrets -n default
```

The deployments must reference the image pull secret.

Example:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: acr-secret
```

Apply this configuration to the relevant deployment manifests.

> An alternative is to grant the AKS cluster identity the appropriate ACR pull permission, which avoids maintaining registry credentials in a Kubernetes secret.

## Step 13: Automate Kubernetes Manifest Updates with Bash

![alt text](<screenshots/script to update the deployment files.png>)

This is the integration point between Continuous Integration and Continuous Delivery.

After an image is built and pushed to ACR, the pipeline needs to update the corresponding Kubernetes manifest with the new image tag.

The Bash script is stored in:

```text
scripts/updatek8sManifests.sh
```

The pipeline passes three arguments:

```text
$1 = Microservice name
$2 = ACR image repository
$3 = Image tag
```

Example:

```bash
#!/bin/bash

set -e

REPO_URL="https://dev.azure.com/Ifedayo001/Azure-project/_git/Azure-project"

git clone "$REPO_URL" /tmp/temp_repo

cd /tmp/temp_repo

sed -i "s|image:.*|image: ifedayoazurecicd.azurecr.io/$2:$3|g" \
  "k8s-specifications/$1-deployment.yaml"

git add "k8s-specifications/$1-deployment.yaml"

git commit -m "Update $1 image to tag $3"

git push

rm -rf /tmp/temp_repo
```

**Authentication note:** The script above illustrates the manifest update logic. For an actual pipeline, authenticate Git operations using a securely supplied Azure DevOps access token, such as `$(System.AccessToken)`, rather than embedding a PAT in the repository URL.

The pipeline task can pass the token through an environment variable:

```yaml
- task: ShellScript@2
  inputs:
    scriptPath: 'scripts/updatek8sManifests.sh'
    args: 'result $(imageRepository) $(tag)'
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

The script can then authenticate the clone using:

```bash
git -c http.extraheader="AUTHORIZATION: bearer $SYSTEM_ACCESSTOKEN" \
  clone "$REPO_URL" /tmp/temp_repo
```

Ensure the pipeline's Build Service identity has the necessary repository permissions, including read and contribute access.

The script updates the relevant deployment manifest and pushes the change to Azure Repos.

For example, a Result pipeline build updates:

```text
k8s-specifications/result-deployment.yaml
```

The image reference changes to:

```text
ifedayoazurecicd.azurecr.io/resultapp:<BUILD_ID>
```

Because Argo CD monitors the same repository, it detects the updated manifest and synchronizes the deployment with AKS.

## Step 14: Verify the Continuous Delivery Process

After the CI pipelines and Argo CD application have been configured, test the complete deployment workflow.

### 14.1 Verify Kubernetes Pods

![alt text](<screenshots/all pods running.png>)

Run:

```bash
kubectl get pods
```

All application pods should eventually show:

```text
STATUS: Running
```

Check the services:

```bash
kubectl get services
```

Check the deployments:
```bash
kubectl get deployments
```

### 14.2 Verify Argo CD

Open the Argo CD web interface.

Confirm that:

- The application is synchronized.
- The application health status is healthy.
- The Kubernetes resources are visible.
- The application pods are running.

### 14.3 Verify Automatic Deployment

Make a change to one of the microservices and push it to Azure Repos.

The expected sequence is:

1. Azure DevOps detects the change.
2. The relevant pipeline starts.
3. Docker builds the updated image.
4. The image is pushed to ACR.
5. The Bash script updates the Kubernetes manifest.
6. The updated manifest is committed to Azure Repos.
7. Argo CD detects the repository change.
8. Argo CD synchronizes the updated deployment with AKS.
9. Kubernetes pulls the new image and updates the application.

This confirms that the CI and CD processes are integrated.

## Step 15: Access the Voting Application

The Vote and Result applications were exposed through Kubernetes NodePort services.

The configured application ports are:

| Application | Access Port |
|---|---:|
| Vote | 30000 |
| Result | 30001 |

Azure inbound network rules were configured to allow access to the application NodePorts.

Open the applications in a browser:

**Vote Application**

```text
http://<NODE_PUBLIC_IP>:30000
```

**Result Application**

```text
http://<NODE_PUBLIC_IP>:30001
```

Replace `<NODE_PUBLIC_IP>` with the appropriate reachable node IP address.

The Vote application allows users to submit votes, while the Result application displays the processed voting results.

> These endpoints use HTTP for this demonstration. A production deployment should use an ingress controller or gateway with HTTPS and appropriate network restrictions.

# Project Challenges and Solutions

During implementation, I encountered some challenges that helped me understand the interaction between Azure DevOps, ACR, AKS, and Argo CD.

| Challenge | Cause | Solution |
|---|---|---|
| AKS pods showed `ImagePullBackOff` | AKS could not authenticate to the private ACR. | Created an image pull secret and referenced it in the Kubernetes deployments. |
| Argo CD was not externally accessible | The service was not exposed through an accessible NodePort and network rule. | Changed the service to NodePort and configured the required Azure inbound rule. |
| CI and CD needed to work together | Building an image alone does not update the Kubernetes deployment. | Created a Bash script to update and push the image tag in the Kubernetes manifest repository. |
| Pipeline Git operations required authentication | The pipeline needed permission to clone and push manifest changes. | Configured repository access for the pipeline identity and used secure token authentication. |

## Video Demonstrations

- **CI Pipeline Demonstration:**

https://github.com/user-attachments/assets/6dc39866-63c5-4936-8e17-bfe0caf6ee36

- **CD / Argo CD Demonstration:**

https://github.com/user-attachments/assets/18cfb36b-f175-4e77-b916-708a669f53b1

# Key Learnings

Through this project, I gained practical experience with:

- Building CI pipelines for multiple microservices.
- Configuring and managing self-hosted Azure DevOps agents.
- Building and pushing Docker images to a private container registry.
- Deploying containerized applications to AKS.
- Configuring Kubernetes services and deployments.
- Implementing GitOps with Argo CD.
- Automating Kubernetes manifest updates with Bash.
- Troubleshooting image pull authentication and Kubernetes deployment issues.
- Understanding how CI and CD work together to automate application delivery.

# Conclusion

This project demonstrates an end-to-end CI/CD and GitOps implementation using Azure DevOps, Azure Container Registry, Azure Kubernetes Service, and Argo CD.

Azure DevOps handles the Continuous Integration process by building and publishing Docker images. The Bash automation script updates the Kubernetes manifests in Azure Repos, while Argo CD continuously monitors the repository and synchronizes the desired application state with AKS.

The result is an automated deployment workflow where application changes can move from source code to a running Kubernetes environment with minimal manual intervention.

## References

- **Original project source:** [Docker Example Voting App](https://github.com/dockersamples/example-voting-app)
- **Azure DevOps:** https://learn.microsoft.com/en-us/azure/devops/
- **Azure Container Registry:** https://learn.microsoft.com/en-us/azure/container-registry/
- **Azure Kubernetes Service:** https://learn.microsoft.com/en-us/azure/aks/
- **Argo CD:** https://argo-cd.readthedocs.io/

---

**Author:** Ifedayo Abidoye

**Project:** Azure DevOps CI/CD and GitOps Deployment of a Microservices Voting Application