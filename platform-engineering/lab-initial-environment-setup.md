# Lab - Initial Environment Setup

In this lab, we will create and setup environment for this training and install all the required tools.

### Learning Objectives

1. Download and install Visual Studio Code and required plugins
2. Download and install Docker (For Containerisation)
3. Download and install Terraform (For Infrastructure as code)
4. Download and install Google Cloud CLI to interact with Google Cloud and provision infrastructures like Kubernetes Cluster, Virtual Machines, Networks, etc.
5. Download and install Kubectl. This CLI utility is required to interact with Kubernetes cluster.
6. Download and install ArgoCD for Kubernetes Deployment

### Steps to follow -

1. **Download and install Visual Studio Code and required plugins**

On the DaDekstop provided by NobleProg team, VS Code is already installed. In case you wish to set it up for your own machine, downloand install from here - [Download VS Code](https://code.visualstudio.com/download)

To open the VS Code on DaDesktop, click on the menu button on top left corner and search for VS Code.

or

Open the terminal and type "code". This will also open the VS Code.



2. **Download and install Docker (For Containerisation)**

On linux machine you can install a Docker Client from here - [Docker Install](https://docs.docker.com/engine/install/ubuntu/)

For all other UI based platforms,  Docker comes as a UI tool called Docker Desktop and can be downloaded and installed from here - [Docker Desktop](https://www.docker.com/products/docker-desktop/)<br>

_Make sure to check the pre-requisites before installing Docker_

On the DaDesktop machine, Docker is already installed, to check if the Docker is installed or not, open terminal and type&#x20;

```
docker --version
```

3. Download and install Terraform&#x20;

Download and install terraform from here - [Terraform Download](https://developer.hashicorp.com/terraform/install)

Check the version of terraform installed on the DaDesktop machine run the following command.

```
terraform --version
```

#### Install Visual Studio Code Extension for Terraform

1. Navigate to the extension tab as highlighted in the screenshot below.

<figure><img src="../.gitbook/assets/unknown (27).png" alt=""><figcaption></figcaption></figure>

2. Search for the terraform extension.

<figure><img src="../.gitbook/assets/unknown (28).png" alt=""><figcaption></figcaption></figure>

Install the extension.<br>

<figure><img src="../.gitbook/assets/unknown (29).png" alt=""><figcaption></figcaption></figure>

Setup is done and we are ready to write Terraform code.

4. Download and install Google Cloud CLI

Download and install Google Cloud CLI from here - [Google Cloud CLI download](https://cloud.google.com/sdk/docs/install#linux)

To check the installation of the Google Cloud CLI, run the following command.

```
gcloud --version
```

5. Download and install ArgoCD&#x20;

Use the curl option from this link to download and install ArgoCD - [Download and install ArgoCD](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

To check the installation of ArgoCD, run the following command -

```
argocd --version
```

6. Download and install **kubectl** command line utility to interact with kubernetes cluster.

Download the kubectl utility from here - [Download kubectl utility](https://kubernetes.io/docs/tasks/tools/)

To check the installation, run the following command -

```
kubectl version --client
```
