---
layout: default
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

# A goat in the boat: a look at how Defender for Containers protect your clusters

<img src="images/kuby-logo.png" style="align: center; margin: 5px; width: 65%;height: auto;" alt="Defender for Containers" >
<span style="display: block;">Source: (c) original picture (without the intruder) is from Matt Butcher</span>

<p></p>
 
<p style="color:#145DA0;">The ubiquitous soar of Kubernetes and containarized or "cloud-native"workloads over the past years has led to an important growth of the security & threat landscape as the number and sophistication of attacks targeting cloud-native environment is booming. While container and Kubernetes security can be hard and requires security practitionners to update their skillsets, a bunch of tools and products have rised in the last years to target these new threat vectors and cope with the elasticity and scalibility of these new workloads. Some of these products include Aquasecurity Trivy, Qualys, Clair, Anchore, Snyk and, the one we will investigate in this post, Microsoft Defender for containers. </p>

<p>
Defender for containers is a recent addition to the (Microsoft Defender for Cloud)[https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction] portfolio, which is a tool for security posture management and threat protection for multi-cloud and hybrid scenarios. Defender for containers is not really 'new', it is an evolution and merge of two previous plans in the defender portfolio: defender for container registries and defender for Kubernetes. This new plan is inline with Microsoft's vision of end-to-end container security: dividing the security of the platform from the security of the container images leads to gaps in a sound container security strategy. </p>

<p>
In this article, we will explore and test Defender for Containers against a vulnerable environment and see what it can detects or prevent but also what are the current limitations compared to other solutions on the market. </p>


## What's with the goat?


  
## What Defender for containers brings?

The idea of this article is not to explore or market Defender for containers, all details can be found in the official documentation. Rather, we will describe here the main features of the product and what they mean in terms of deployment/usage. 

