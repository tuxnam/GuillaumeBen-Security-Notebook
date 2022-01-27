---
layout: default
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

# A goat in the boat: a look at how Defender for Containers protects your clusters

<p style="width: 100%; text-align: center;">
<img src="images/kuby-logo.png" style="align: center; margin: 5px; width: 65%;height: auto;" alt="Defender for Containers" >
<span style="display: block; font-size: 12px;">Source: original picture (without the intruder) is from <a href="https://www.cncf.io/phippy/#:~:text=The%20Illustrated%20Children%E2%80%99s%20Guide%20to%20Kubernetes%20The%20Illustrated,try%20to%20explain%20software%20engineering%20to%20their%20children.">Matt Butcher and his illustrated guide of Kubernetes</a></span>
</p>
 
<p style="color:#145DA0;">The ubiquitous soar of Kubernetes and containarized or "cloud-native" workloads over the past years has led to an important growth of the security & threat landscape as the number and sophistication of attacks targeting cloud-native environment is booming. While container and Kubernetes security can be hard and requires security practitionners to update their skillsets, a bunch of tools and products have rised in the last years to target these new threat vectors and cope with the elasticity and scalibility of these new workloads. Some of these products include Aquasecurity Trivy, Qualys, Clair, Anchore, Snyk and, the one we will investigate in this post, Microsoft Defender for containers. </p>

***In this article, we will explore and test Defender for Containers against a vulnerable environment and see what it can detects or prevent but also what are the current limitations compared to other solutions on the market.***

<p>
Defender for containers is a recent addition to the <a href="https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction">Microsoft Defender for Cloud</a> portfolio, which is a tool for security posture management and threat protection for multi-cloud and hybrid scenarios. Defender for containers is not really 'new', it is an evolution and merge of two previous plans in the defender portfolio: defender for container registries and defender for Kubernetes. <br />
This new plan is inline with Microsoft's vision of end-to-end container security: dividing the security of the platform from the security of the container images leads to gaps in a sound container security strategy. </p>


## Content of this article

1. Environment setup
2. Exploring Defender for Containers
3. Enabling Defender 
4. Images vulnerabilities
5. First look at cluster alerts
6. Going through the goat scenarios and how Defender reacts
7. Using policies to resolve/prevent weaknesses
8. A note on CI/CD integration

## The setup and the goat story

In order to have a valuable testing environment, we need to have a cluster of course, a container registry, container images but most importantly weaknesses, exploit them and see how Defender for containers can help. <br />
The demo environment we will leverage will therefore be the following:
- One Azure Kubernetes cluster (could be any CNCF-compliant cluster)
- One Azure Container registry 
- Defender for containers enabled for the target resources
- One GitHub kiss (keep-it-simple-stupid) pipeline
- A set of vulnerable workloads

The vulnerable workload, aka the goat in the boat, will be leveraging the awesome work of a cloud-native security specialist, Madhu Akula. <a href="https://github.com/madhuakula/kubernetes-goat">Kubernetes Goat</a> is designed to be an intentionally vulnerable cluster environment to learn and practice Kubernetes security.
We will deploy Kubernetes Goat on our AKS cluster, go through the scenarios and see how Defender is helping to detect/mitigate related weaknesses and threats.
<br />
<br />
Kubernetes Goat is originally pulling container images used in the various scenarios from Madhu's own Docker repository. For the sake of this exercice, the ability to test the vulnerability scanning features of Defender for containers, and the principle to always deploy from a trusted registry, I modified the coressponding deployment files of Kubernetes Goat to take the same container images but from the container registry we have set up for this lab. *Thanks to Madhu for his work!*
<p></p>
Here is an overview of the setup with deployed namespaces: <br />
<img src="images/Topology-goat-aks.png" style="float: center; align: center;" alt="Defender for Containers environment setup" >
  
## Defender for Containers  
  
### What does it brings?

The idea of this article is not to explore or market Defender for containers, all details can be found in the official ![documentation](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction?tabs=defender-for-container-arch-aks) or this ![article](https://techcommunity.microsoft.com/t5/microsoft-defender-for-cloud/introducing-microsoft-defender-for-containers/ba-p/2952317). 
<br /> Rather, we will describe here the main features of the product and what they mean in terms of deployment/usage. 

<p style="width: 100%; text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151179154-9d313434-c093-4303-85e3-744a6065b55c.png" />
</p>

In summary, defender for containers provides:
- Environment hardening for any <a href="https://www.cncf.io/certification/software-conformance/">CNCF compliant cluster(s)</a> (AKS, EKS, Kubernetes IaaS, Openshift...): visibility into misconfigurations and guidelines to help mitigate identified threats
- Run-time threat protection for nodes and clusters: Threat protection for clusters and Linux nodes generates security alerts for suspicious activities
- Vulnerability scanning: Vulnerability assessment and management tools for images <u>stored</u> in Azure Container Registries (support for third-party registries soon) and <u>running</u> in Kubernetes.

### How does it bring it?

The sources analyzed by Defender are multiple:
- Audit logs and security events from the API server
- Cluster configuration information from the control plane
- Workload configuration from Azure Policy
- Security signals and events from the node level

In terms of deployment, that means:
A Defender profile deployed to each node provides the runtime protections and collects signals from nodes using eBPF technology: it relies on a <a href="https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/">DaemonSet</a> to ensure all nodes in the cluster are covered.
Azure Policy add-on for Kubernetes collects cluster and workload configuration for admission control policies (Gatekeeper): this basically translates Azure Policies to Gatekeeper/OPA policies (see <a href="https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes#:~:text=Azure%20Policy%20extends%20Gatekeeper%20v3%2C%20an%20admission%20controller,state%20of%20your%20Kubernetes%20clusters%20from%20one%20place.">here</a> for details on OPA and Gatekeeper) for audit, enforcement and prevention of policies on the admission controller of your clusters.<br />
The container registry does not need anything specific, outside of network considerations.

## Step 1: cluster is ready, the goat is in the boat...let's enable Defender and look at the cluster

## Step 2: are there some image vulnerabilities?

## Step 3:


