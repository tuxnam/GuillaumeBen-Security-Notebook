---
layout: post
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

#### Last update: April 2022

# Automated incident response on Kubernetes: leverage Azure SOAR capabilities to respond to Kubernetes security alerts
<p></p>
<span class="subtitle">In this article, I am describing and sharing playbooks which allow a SOC analyst to perform incident response on a target Kubernetes cluster, in Microsoft Sentinel, in a response to an alert raised by Microsoft Defender for Cloud or from Defender for Cloud directly. 
<p></p>

<p style="width: 100%; text-align: center;">
<img src="images/AKS-resp-logo.png" style="align: center; margin: 5px; width: 65%;height: auto;" alt="DFIR on AKS" >
<p></p>
</p>
 
> As discussed in my [previous article](http://guillaumeben.xyz/defender-containers.html) on Defender for Containers, the skyrocketting usage of Kubernetes and containarized workloads over the past years has led to new attack vectors. We can expect these attacks to grow in number and sophistications in the future, and be also (more) leveraged by nation-state actors and in advanced persistent threats.<br /> 
> One of the recurring issue for Security Operations teams is to be able not only to monitor critical Kubernetes environment in their respective organizations, have the ability, knowledge and tools to be able to detect a security incident on Kubernetes, but also respond adequately or collect evidence.
> Incident Response on Kubernetes can range fron artifacts collection and labelling to pod or namespace isolation. 
> I am describing in this article a solution to automate incident response on AKS, from Defender for Cloud or Microsoft Sentinel. 
> The solution will work natively with alerts raised by Defender for Containers, but can be adapted for your own alerts and detection capabilities in Azure.
 <p></p>

---
*DISCLAIMER: This solution and the code therein are meant to be run in a development or testing environment first and adapted to your own needs. The python code used for the Azure Function could benefit some refactoring and may lead to errors or exceptions in specific cases. The current set of supported scenarios is limited and meant to work with alerts raised by Defender for Cloud or alerts having entities formatted the same way (see corrsponding specification section). Current features, limitations and next steps are described here. Any idea or suggestion is welcomed as a GitHub Issue ().*
---

## Description of the solution

The solution's goal is to perform automated reponse on alerts raised by Defender for Containers and leverages the following assets:
* An Azure Function written in Python
* A set of logic apps (playbooks) for each supported response 
* A BLOB storage to upload collected artifacts (Optional)

### Authentication and security

The Azure Function leverages an [Azure Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) to issue command and response actions to the targeted AKS cluster. This allows to avoid using or managing static credentials. <br />
The function also allows to upload targeted artifacts to a pre-existing BLOB storage, relying on SAS token which can be short-lived. <br />
The function and the logic apps should all enforce TLS 1.2.<br />
<br />
Malicious access to the function could lead to a full compromise of your AKS cluster. Managed identity should thus be controlled carefully.

### List of supported responses

The solution currently allows for analysts to trigger the following responses, as playbook:
- Label and isolate a specific pod on targeted cluster
- Label and isolate a complete namespace on targeted cluster
- Label and cordon a AKS node on targeted cluster
- Collect pod artifacts and upload them to a BLOB storage on targeted cluster
- Run a command on a pod on targeted cluster

All operations can be reverted (uncordon, remove isolation...)

## Specifications and usage

### API specification for HTTP triggered Azure Function

The format used by the Azure function as input to trigger a response is based on the JSON format of entities part of alerts raised by Defender for Containers.<br />
<br />
**Example:**<br /><br />

```
{
  "action": "<ACTION>",
  "entities": [
    {
      "$id": "2",
      "ImageId": "<CONTAINER_IMAGE>",
      "Type": "container-image"
    },
    {
      "$id": "3",
      "Name": "<CONTAINER_NAME>",
      "Image": {
        "$ref": "2"
      },
      "Type": "container"
    },
    {
      "$id": "4",
      "ResourceId": "/subscriptions/<SUBSCRIPTION_ID/resourceGroups/<RG_NAME>/providers/Microsoft.ContainerService/managedClusters/<CLUSTER_NAME>",
      "Type": "azure-resource"
    },
    {
      "$id": "5",
      "CloudResource": {
        "$ref": "4"
      },
      "Type": "K8s-cluster"
    },
    {
      "$id": "6",
      "Name": "<NAMESPACE-NAME>",
      "Cluster": {
        "$ref": "5"
      },
      "Type": "K8s-namespace"
    },
    {
      "$id": "7",
      "Name": "<DEPLOYMENT-NAME>",
      "Namespace": {
        "$ref": "6"
      },
      "Type": "K8s-deployment"
    },
    {
      "$id": "8",
      "Name": "<POD-NAME>",
      "Namespace": {
        "$ref": "6"
      },
      "Type": "K8s-pod"
    }
  ],
  "storageaccount":"<STORAGE-ACCOUNT-NAME>",
  "blobcontainer":"<BLOB-CONTAINER-NAME>",
  "blobsastoken":"<BLOB-SAS-TOKEN>"
}
```

## Example usage

### From Defender for Cloud: automation on alerts from Defender for Containers

### From Sentinel: run a playbook as a response to an alert


## Installation

### Azure Function

### Playbooks 

#### Collect Artifacts Playbook 

**Specification:**<br />
This playbook allows to collect the following artifcats from a pod entity contained in a Defender for Container alert from Sentinel:
- Pod logs (kubectl logs <pod>)
- Pod full description (kubectl get <pod> -o json)
- Pod description (kubectl describe <pod>)

**Required Parameters:**<br />
 
- Playbook Name: name you want to give to this playbook
- Username: username for the connection of the playbook to Sentinel 
- Storage Account: storage account name used to collect artifacts
- BLOB Container Name: BLOB container name inside the storage account
- SAS Token URI: the SAS token to be used to upload to the BLOB storage (make sure it is short-lived)
- AKS Response Function name: the name of the related AKS response function part of this solution
- AKS Response Functio code: the access key used to authenticate and call the Azure Function
 
**Expected Entities in Sentinel Alert:** <br />

 The minimum list of entities required in the JSON body of the alert is the following (can contain more entities, which will just be ignored):
 
 ```
  [
    {
      "ResourceId": "/subscriptions/6178a2ae-xxxx-xxxx-xxxx-xxx95af392b9/resourceGroups/k8s-demo-rg/providers/Microsoft.ContainerService/managedClusters/my-super-cluster",
      "Type": "azure-resource"
    },
    {
      "Name": "damn-vuln-cluster",
      "Type": "K8s-cluster"
    },
    {
      "Name": "default",
      "Type": "K8s-namespace"
    },
    {
      "Name": "health-check-deployment-7d78966d57-gjk49",
      "Type": "K8s-pod"
    }
  ]
 ```
 
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://raw.githubusercontent.com/tuxnam/Azure-AKS-Incident-Response/main/LogicApps/AKS-Resp-CollectArtifacts/azuredeploy.json?token=GHSAT0AAAAAABOR6J3GM4WRI7H65LZZRHEAYTYZSHA)

#### Isolate Pod

#### Isolate Namespace

#### Cordon Node

#### Run Command



All the code used in this solution can be found here: .

1. The setup of the function can be done using the following link. This will deploy a Python Azure Function. 

2. Assign roles to the managed identity of the Azure Function

3. Playbooks 

Microsoft Sentinel and Defender for Cloud playbooks can be imported by using the following link: 

4. BLOB Storage (optional)

## Usage

## Limitations and next steps
