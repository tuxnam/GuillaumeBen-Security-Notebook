---
layout: post
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

#### Last update: April 2022

# Automated incident response on AKS: leverage Sentinel SOAR capabilities to respond to Kubernetes security issues
<p></p>
<span class="subtitle">In this article, I am describing and sharing playbooks which allow a SOC analyst to perform incident response on a target Kubernetes cluster, in Microsoft Sentinel, in a response to an alert raised by Microsoft Defender for Cloud or from Defender for Cloud directly. Automatically or with a manual trigger, depending on the analyst needs. </span>
<p></p>

<p style="width: 100%; text-align: center;">
<img src="images/AKS-resp-logo.png" style="align: center; margin: 5px; width: 65%;height: auto;" alt="DFIR on AKS" >
<p></p>
</p>
 
> As discussed in my [previous article](http://guillaumeben.xyz/defender-containers.html) on Defender for Containers, the skyrocketting usage of Kubernetes and containarized workloads over the past years has led to new attack vectors. 
> One of the recurring issue for Security Operations teams is to be able not only to monitor critical Kubernetes environment in their respective organizations, have the ability to detect a security incident on Kubernetes, but also there is a need to respond.
> Incident Response on Kubernetes can range fron artifacts collection, to pod or namespace isolation. 
> I am describing in this article a solution to automate incident response on AKS, from Defender for Cloud or Microsoft Sentinel. 

 <p></p>

<p style="background-color: #FFFFFF;">
This solution and the code therein are meant to be tested in a development or testing environment first and adapted to your own needs.<br />
The python code used for the Azure Function could benefit some refactoring and may lead to errors or exceptions in specific cases.
</p>

## Description of the solution

The solution leverages the following assets:
* An Azure Function written in Python 
* A set of logic apps based on the required response 
* A BLOB storage to upload collected artifacts

### Authentication and security

The Azure Function leverages a [Azure Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) to issue command and response actions to the targeted AKS cluster. This allows to avoid using or managing static credentials. <br />
The function also allows to upload targeted artifacts to a pre-existing BLOB storage, relying on SAS token. <br />
The function and the logic apps should all enforce TLS 1.2.<br />

### List of supported responses

The solution currently allows for analysts to trigger the following responses:
- Label and isolate a specific pod on targeted cluster
- Label and isolate a complete namespace on targeted cluster
- Label and cordon a AKS node on targeted cluster
- Collect pod artifacts and upload them to a BLOB storage on targeted cluster
- Run a command on a pod on targeted cluster

All operations can be reverted (uncordon, remove isolation...)

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

### Detailed API specification


## Example usage

### From Defender for Cloud: automation on alerts from Defender for Containers

### From Sentinel: automation rule and playbooks


## Installation

All the code used in this solution can be found here: .

1. The setup of the function can be done using the following link. This will deploy a Python Azure Function. 

2. Assign roles to the managed identity of the Azure Function

3. Playbooks 

Microsoft Sentinel and Defender for Cloud playbooks can be imported by using the following link: 

4. BLOB Storage (optional)

## Usage

## Limitations and next steps
