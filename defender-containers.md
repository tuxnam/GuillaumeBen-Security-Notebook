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

**Note:** In my case, I ued _kubenet_ for kubernetes network driver and _Calico_ for network policies but it does not matter in this context and is not needed either.
  
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

So after setting up Kubernetes (standard set up, two nodes)(you can refer [here](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal#:~:text=To%20create%20an%20AKS%20cluster%2C%20complete%20the%20following,create%20an%20Azure%20Resource%20group%2C%20such%20as%20myResourceGroup) for Azure AKS), and deploying the Kubernetes Goat (refer to Madhu's documentation [here](https://madhuakula.com/kubernetes-goat/)), we enabled Defender for Containers with auto-provisionning (see documentation for details), in Defender for Cloud portal:

![image](https://user-images.githubusercontent.com/18376283/151562065-52e0cee8-b3d3-4614-b2de-a63c248ffdae.png)

**Note:** _Arc-enabled cluster_ is for Kubernetes clusters on IaaS in another Cloud or on your premises. 
Details on setup for all type of clusters, including EKS, can be found [here](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-enable?tabs=aks-deploy-portal%2Ck8s-deploy-asc%2Ck8s-verify-asc%2Ck8s-remove-arc%2Caks-removeprofile-api&pivots=defender-for-container-aks).

let's have a first look at our Kubernetes cluster and the impact of enabling Defender: we can clearly see the Defender Profile related-pods (deployed through the Daemonset) and the Gatekeeper namespace/pods, next to the Kubernetes Goat related namespaces and pods:

![image](https://user-images.githubusercontent.com/18376283/151561842-f1036784-8fe6-4e71-bdd8-6878aa339946.png)


## Step 2: are there some image vulnerabilities?

Earlier in this article, we uploaded the various container images required for Kubernetes Goat into our Azure Container Registry. Here is what it looks like in terms of repositories:

![image](https://user-images.githubusercontent.com/18376283/151572088-8d7b0994-0788-4219-b169-875d32223540.png)

Since we enabled Defender, and one of the features is vulnerability scanning of container images pushed, pulled and recently pulled in your registry, let's have a look at the results. For this, we simply go in Defender for Cloud portal, 'Workload Protection': we can there at the same time see our Kubernetes cluster inside that subscription are covered.

![image](https://user-images.githubusercontent.com/18376283/151572496-050f3956-bd78-40f5-874b-f9d36142e772.png)

If we click on _Container Image Scanning_ we can see that Defender indeed scanned our images (using Qualys behind the scenes) and is already giving us a list of CVEs to tackle: 

![image](https://user-images.githubusercontent.com/18376283/151572831-3c15c6de-9f01-4b04-a535-07c69167cc17.png)

These CVEs means that the Goat container images need some updates, to avoid introducing (yet more) vulnerabillties in our environment. You can click on a registry to see all details about affected images and tags:

![image](https://user-images.githubusercontent.com/18376283/151573228-c28541e2-4e47-4911-9d8b-d0280be0c0a8.png)
![image](https://user-images.githubusercontent.com/18376283/151573373-cb19632b-4365-4736-aade-8f5869dce44b.png)

## Ok, but we deployed a goat, did Defender detect something?

So, we deployed KKubernetes Goat on the cluster as you saw from the various pods listed here above in the printscreen. However, Goat is meant to be intentionnally vulnerable!  Even if we did not start the scenarios yet, did Defender already catch some issues with this deployment?
Let's jump to security alerts in the same Defender for Cloud portal and filter on our cluster:

![image](https://user-images.githubusercontent.com/18376283/151575616-66b708d4-4967-4ff3-9526-01163e82cb07.png)

We notice 3 alerts, after the setup:
- Two privileged containers running
- One container with sensitive mount

If we check details of one of these alerts, we can see interesting information, including related [MITRE tactics](https://www.microsoft.com/security/blog/2021/04/29/center-for-threat-informed-defense-teams-up-with-microsoft-partners-to-build-the-attck-for-containers-matrix/), detailed description, pod name...

![image](https://user-images.githubusercontent.com/18376283/151575909-b61d9f49-ce81-484c-8430-8e2c7acf62c0.png)

However we would like to potentially trigger actions based on this alert or have information about how to mitigate the threat! This is all in the 'take action' tab:

![image](https://user-images.githubusercontent.com/18376283/151576116-b7b8d355-8140-419b-a7de-dfe93633436d.png)

1. We can see some detailed advise to mitigate the threats
2. We have a list of ***actionable*** recommendations to prevent this threat in the future!
What are these mitigations exactly? They are leveraging Gatekeeper which we briefly discussed earlier, the admission controller, and Azure Policy, to enforce controls and best-practices on your clusters! Azure Policy extends Gatekeeper v3, the admission controller webhook for Open Policy Agent (OPA), to apply at-scale enforcements and safeguards on your clusters in a centralized, consistent manner. Azure provides built-in policies such as the ones proposed in the alert remediation, or yet Kubernetes CIS hardening policies but you can also come up with custom policies. <br />
You are able to audit, prevent or remediate issues in your cluster in an automated way. We will not do this here, as this would break our vulnerable goat of course, but let's keep this in mind for the end of this article. 

All details about Gatekeeper and Azure Policy can be found [here](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes#:~:text=Azure%20Policy%20extends%20Gatekeeper%20v3%2C%20an%20admission%20controller,state%20of%20your%20Kubernetes%20clusters%20from%20one%20place.).

### Nut did Defender detect all potential issues and weaknesses?

Only three alerts? Uh. But I thought the goat was damn vulnerable! Good catch! But it is not only about alerts, also about recommendations! In the above list of recommendations, in alert details, let's click on 'view all recommendations'...surprisethere is way more problems in our safe Kubernetes environment than we thought! 

![image](https://user-images.githubusercontent.com/18376283/151578380-91e489be-b017-4577-ba1e-3ad444a9c588.png)

We see clearly here that these rules are in 'audit' mode, we could, through the same Azure Policy for Gatekeeper feature described, enforce or remediate these issues. Have a look at corresponding documentation!

We can also see of course a few recommendations are 'met' and compliant:

![image](https://user-images.githubusercontent.com/18376283/151578774-bc5ca6fb-18c9-4343-bed7-455c1fc1d0b7.png)

You do not have to go or have to wait for an alert to see these best-practice issues.
They are listed as recommendations, amongst others in your Defender for Cloud Security Posture Management tab:

![image](https://user-images.githubusercontent.com/18376283/151579510-361fa0e4-9ffb-476a-8f81-7572f3214498.png)

The full list of detection capabilities (up-to-date) of Defender can be found here: https://docs.microsoft.com/en-us/azure/defender-for-cloud/alerts-reference#alerts-k8scluster
The list of recommendations can be found here: https://docs.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference


### Wait...we saw CVEs in container images in the registry, isn't Defender also supposed to alert me that these vulnerable images are now running in my clusters?

Good catch! Indeed, it should also be part of the reommendation but is a preview feature as time of writing, details here: https://docs.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference

## Let's dive into the first scenario's of Kubernetes Goat

Now, let's play and see how Defender reacts. Details about the scenarios of the Kubernetes Goat can be found [here](https://madhuakula.com/kubernetes-goat/about.html#:~:text=Kubernetes%20Goat%20is%20designed%20to%20be%20an%20intentionally,production%20environment%20or%20alongside%20any%20sensitive%20cluster%20resources.).
We will not expand too much on the details here, for the sake of your not falling asleep, but also because Madhu desceibes already everything you should know in his documentation, and in some good videos such as Defcon talks. Also, the idea is not to spoil solutions and try to capture the flag by yourself!

The first scenario is about secrets hidden in plainsights! No specific interesting triggers for Defender here, rather good CI/CD pipeline hygiene to have, and a recurring issue in codebases.

Scenario 2 looks more interesting (for our case :)) and seems to be related to container escape! Let's try it and see what Defender brings. The first thing to notice is that this scenario is linked to container image *system-monitor*. Why does it matter? because if you remember, Defender raised 3 alerts after we deployed the cluster and one of them was the following:

![image](https://user-images.githubusercontent.com/18376283/151582721-b345255d-46d6-49ef-9e47-1b83c4aa8be3.png)

Let's ignore the alert and still do the scenario. What will Defender do?





