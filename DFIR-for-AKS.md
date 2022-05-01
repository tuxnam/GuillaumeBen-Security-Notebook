---
layout: post
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

#### Last update: April 2022

# Automated incident response on AKS: leverage Azure SOAR capabilities to respond to Kubernetes security issues
<p></p>
<span class="subtitle">In this article, I am describing and sharing playbooks which allow a SOC analyst to perform incident response on a target Kubernetes cluster, as part of an incident investigation in Microsoft Sentinel or in a response to an alert raised by Microsoft Defender for Cloud, for instance. </span>
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

## Current Status and Limitations

## Description of the solution

### List of Supported Responses
