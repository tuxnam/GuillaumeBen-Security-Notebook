---
layout: post
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

# A goat in the boat: a look at how Defender for Containers protects your clusters
<p></p>
<span class="subtitle">In this article, we will explore and test Defender for Containers against a vulnerable environment and see what it can detects or prevent and how we can leverage it to make our Kubernetes workloads safer.</span>
<p></p>

<p style="width: 100%; text-align: center;">
<img src="images/kuby-logo.png" style="align: center; margin: 5px; width: 65%;height: auto;" alt="Defender for Containers" >
<span style="display: block; font-size: 12px;">Source: original picture (without the intruder) is from <a href="https://www.cncf.io/phippy/#:~:text=The%20Illustrated%20Children%E2%80%99s%20Guide%20to%20Kubernetes%20The%20Illustrated,try%20to%20explain%20software%20engineering%20to%20their%20children.">Matt Butcher and his illustrated guide of Kubernetes</a></span>
 <p></p>
</p>
 
> The skyrocketting usage of Kubernetes and containarized workloads over the past years has led to new attack vectors. 
> The number and sophistication of attacks targeting cloud-native environment is booming. While containers and Kubernetes security can be hard and require security professionals to update their skillsets, a bunch of tools and products have rised to target these new threats and cope with the elasticity and scalibility needs of Kubernetes workloads. 
> Some of these products include Trivy, Qualys, Clair, Anchore, Snyk, a myriad of good open-source tools and, the one we will investigate in this post, Microsoft Defender for containers. 

 <p></p>

## What is Defender for Containers?

Defender for containers is a recent addition to the <a href="https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction">Microsoft Defender for Cloud</a> portfolio, which is a tool for security posture management and threat protection for multi-cloud and hybrid scenarios. 
Defender for containers is not really 'new', it is an evolution and merge of two previous plans in the defender portfolio: defender for container registries and defender for Kubernetes. <br />
This evoltuon, however, is inline with Microsoft's vision of end-to-end container security: bridging the gap between the security of the platform (Kubernetes) and the security of the container(s) themselves. Isolating these concepts leads to gaps in a sound container security strategy.


## Content of this article

#### [1. The setup and the goat story](#Item-1)
#### [2. Meet Defender for Containers](#Item-2)
#### [3. Cluster is ready, the goat is in the boat...](#Item-3) 
#### [4. A look at some vulnerabilities](#Item-4) 
#### [5. What about runtime scanning?](#Item-5) 
#### [6. We deployed a goat, did Defender detect something already?](#Item-6) 
#### [7. Testing attack patterns](#Item-7) 
#### [8. Conclusion](#Item-8) 
#### [8. References](#Item-9) 

<p></p>

<a name="Item-1"></a>
## The setup and the goat story

The first element we need is to setup a valuable testing environment, with a simple Kubernetes cluster, a container registry, container images and most importantly weaknesses and the ability to exploit them to see how Defender for containers can help. <br />

The demo environment will therefore consist of the following assets:
- One Azure Kubernetes cluster (could be any CNCF-compliant cluster)
- One Azure Container registry 
- Defender for containers enabled on the target resources
- One GitHub basic pipeline (*optional*)
- A vulnerable workload

### Introducing Kubernetes Goat and detailing the testing environment 

The vulnerable workload, aka the goat in the boat, will be leveraging the awesome work of a DevOps security specialist, Madhu Akula. <br />
<a href="https://github.com/madhuakula/kubernetes-goat">Kubernetes Goat</a> is designed to be an intentionally vulnerable cluster environment to learn and practice Kubernetes security. <br />
We will deploy Kubernetes Goat on our cluster, do some steps of the scenarios proposed, but also we will attempt abusing the vulnerable environment itself and see how Defender can help to detect/mitigate related weaknesses and threats.
<br />
<br />
Kubernetes Goat is originally pulling container images used in the various scenarios from the author's own Docker repository. For the sake of this exercice, the ability to test the vulnerability scanning features of Defender for containers, and the principle to always deploy from a trusted registry, we will adapt the deployment files of Kubernetes Goat to take the same container images but from the container registry we have set up for this lab. 
<br />
<br />
As an example:

<div style="text-align: center; max-width: 70%;">
<img src="https://user-images.githubusercontent.com/18376283/156877420-7c5f41ca-79a5-4b59-bb8b-d742508d332c.png" alt="adapted YAML" />
</div>

...becomes...

<div style="text-align: center; max-width: 70%;">
<img src="https://user-images.githubusercontent.com/18376283/156877538-c73dd7b0-cd52-4a77-85f0-ac60004c00da.png" alt="original YAML" />
</div>

<p></p>

*Thanks to Madhu for his awesome work!*

<p></p>

Here is an overview of the complete setup with deployed namespaces (highlighted the namespaces were pods from k8s goat are deployed): 
<br />
<br />
<div style="text-align: center">
<img src="images/Topology-goat-aks.png" style="float: center; align: center;" alt="Defender for Containers environment setup" >
</div>

<p></p>

**Note:** In my case, I used _Azure CNI_ for kubernetes network driver and _Azure_ for network policies but it does not matter in this context and is not needed either for our use case.
 
<a name="Item-2"></a>
## Defender for Containers 

  
### What does it bring?

The idea of this section is not to explore all features of Defender for containers, all details can be found in the official [documentation](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction?tabs=defender-for-container-arch-aks) or this [article](https://techcommunity.microsoft.com/t5/microsoft-defender-for-cloud/introducing-microsoft-defender-for-containers/ba-p/2952317). 
<br /> Rather, we will describe here the main aspects of the product and what they mean in terms of deployment/usage. 

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151179154-9d313434-c093-4303-85e3-744a6065b55c.png" />
</div>

<p></p>

In summary, defender for containers provides:
- Environment hardening for any <a href="https://www.cncf.io/certification/software-conformance/">CNCF compliant cluster(s)</a> (AKS, EKS, Kubernetes IaaS, Openshift...): visibility into misconfigurations and guidelines to help mitigate identified threats
- Run-time threat protection for nodes and clusters: Threat protection for clusters and Linux nodes generates security alerts for suspicious activities
- Vulnerability scanning: Vulnerability assessment and management tools for images <u>stored</u> in Azure Container Registries (support for third-party registries soon) and <u>running</u> in Kubernetes.

### How does it bring it?

The sources analyzed by Defender are multiple:
- Audit logs and security events from the API server
- Cluster configuration information from the control plane
- Workload configuration from Azure Policy
- Security signals and events from the node/pod

In terms of deployment, this means:<br />
- A Defender profile deployed to each node provides the runtime protections and collects signals from nodes using (eBPF technology)[https://ebpf.io/]: it relies on a <a href="https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/">DaemonSet</a> to ensure all nodes in the cluster are covered.<br />
- A Azure Policy add-on for Kubernetes, which collects cluster and workload configuration for admission control policies (Gatekeeper): this basically translates Azure Policies to Gatekeeper/OPA policies (see <a href="https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes#:~:text=Azure%20Policy%20extends%20Gatekeeper%20v3%2C%20an%20admission%20controller,state%20of%20your%20Kubernetes%20clusters%20from%20one%20place.">here</a> for details on OPA and Gatekeeper) for audit, enforcement and prevention of policies on the admission controller of your clusters.<br />
- The container registry does not need anything specific, outside of network considerations.

<a name="Item-3"></a>
## Cluster is ready, the goat is in the boat...

So after setting up Kubernetes (standard set up, two nodes)(you can refer [here](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal#:~:text=To%20create%20an%20AKS%20cluster%2C%20complete%20the%20following,create%20an%20Azure%20Resource%20group%2C%20such%20as%20myResourceGroup) for Azure AKS), and deploying the Kubernetes Goat (refer to Madhu's documentation [here](https://madhuakula.com/kubernetes-goat/)), we enabled Defender for Containers with auto-provisionning, in Defender for Cloud portal:

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151562065-52e0cee8-b3d3-4614-b2de-a63c248ffdae.png" />
</div>
<p></p>
**Note:** _Arc-enabled cluster_ is for Kubernetes clusters on IaaS in another Cloud or on your premises. 
Details on setup for all type of clusters, including EKS, can be found [here](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-enable?tabs=aks-deploy-portal%2Ck8s-deploy-asc%2Ck8s-verify-asc%2Ck8s-remove-arc%2Caks-removeprofile-api&pivots=defender-for-container-aks).

Let's have a first look at our Kubernetes cluster and the impact of enabling Defender: we can clearly see the Defender Profile related-pods (deployed through the daemonset) (in red) and the Gatekeeper pods (in green), next to the Kubernetes Goat (in gold) related namespaces and pods:

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/155592893-76703234-46e0-4209-831d-a557da469f28.png" />
<img src="https://user-images.githubusercontent.com/18376283/155593066-724848e0-af6d-4a3d-828d-40a28d9e4d9d.png" />
</div>
<p></p>

We can also notice the daemonset we referred to:

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/155593233-21ac32ad-8348-4c24-a922-1898800ca475.png" />
</div>

<a name="Item-4"></a>
## A look at image vulnerabilities

Earlier in this article, we pushed the various container images required for Kubernetes Goat into our container registry. <br />
Here is what it looks like in terms of repositories:

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151572088-8d7b0994-0788-4219-b169-875d32223540.png" />
</div>
<p></p>

Since we enabled Defender, and one of the features is vulnerability scanning of container images, let's have a look at the results. For this, we can navigate to the Defender for Cloud portal, *Workload Protection* tab (at the same time, we can confirm that our Kubernetes clusters inside the target subscription are covered by Defender):

<p></p>

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151572496-050f3956-bd78-40f5-874b-f9d36142e772.png" />
</div>
<p></p>

If we click on _Container Image Scanning_ we can see that Defender indeed scanned our images and is already giving us a list of CVEs to tackle: 

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151572831-3c15c6de-9f01-4b04-a535-07c69167cc17.png" />
</div>

<p></p>
These CVEs means that the Goat container images need some updates, to avoid introducing (yet more) vulnerabillties in our environment. 
You can click on a registry to see all details about affected images and tags:

<div style="text-align: center;">
<img src="https://user-images.githubusercontent.com/18376283/151573228-c28541e2-4e47-4911-9d8b-d0280be0c0a8.png" />
<img src="https://user-images.githubusercontent.com/18376283/151573373-cb19632b-4365-4736-aade-8f5869dce44b.png" />
</div>

<p></p>

**Note**: the detected vulnerabilities are CVEs, or software vulnerabilities in libraries or OS version used in these containers. As of today, defender for containers does not scan for CIS, secrets in container images or similar best-practices. You can however use other tools such as Trivy in order to bring that capability to your pipeline.

### Can I trigger an on-demand scan?

No. As of today, the scans are happening on push, on pull, and recurrently on recently pulled images from your registry (weekly basis). 
You can expect this to evolve with the solution roadmap. <br />
There is however the ability to leverage [Defender for Containers in your CI/CD pipeline, using Trivy](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-container-registries-cicd) and therefore act on the scans results as part of your integration or delivery pipeline, on-demand. <br />
Also note that this scan includes not only CVEs but also best-practice checks.

<a name="Item-5"></a>
## What about runtime scanning?

We deployed already the containers listed as vulnerable in the registry to our AKS cluster, did defender spot that? <br />
Yes! If we go back to our Defender portal, and go through recommendations, there is one specific to runtime container vulnerabilities (in red), next to the container registry ones (in yellow):

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155594158-df3ab3d4-5041-4a4f-af0d-441ebd1aa645.png" />
</div>
<p></p>

When we open this recommendation, we can see the same CVEs as in the container registry scans:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155594301-ad8bdcd6-f493-4e1a-a742-f14e93b973b1.png" />
</div>
<p></p>
 
### Can I prevent vulnerable containers from being deployed?

It would be ideal to, not only have a view on vulnerable containers but also prevent vulnerable containers to be deployed. Thanks to Azure Policy for Gatekeeper and defender for containers, we will be able to do so (soon)! <br />
This is in fact one of the recommendation also made by Defender for Containers about our cluster:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155594817-dcb97ae1-9263-4243-b639-ec0918ff566b.png" />
</div>
<p></p>

This feature is still in preview and will require a mutation admission controller. Today, Gatekeeper supports OPA queries based only on static conditions and data which exists on request body at admission time. You can check the policy and the evolution of that capability [here](https://docs.microsoft.com/en-us/azure/aks/policy-reference#policy-definitions). <br /><br /> 
Azure Policy extends Gatekeeper v3, the admission controller webhook for Open Policy Agent (OPA), to apply at-scale enforcements and safeguards on your clusters in a centralized, consistent manner. Azure provides built-in policies such as the ones proposed in the alert remediation, or yet Kubernetes CIS hardening policies but you can also come up with custom policies. <br />
You are able to audit, prevent or remediate issues in your cluster in an automated way. We will not do this here, as this would break our vulnerable goat of course.
You can check this [page](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes) for Azure Policy for Kubernetes in general.

<a name="Item-6"></a>
## We deployed a goat, did Defender detect something already?

So, we deployed Kubernetes Goat on the cluster as you saw from the various pods listed here above in the printscreen. 
However, Goat is meant to be intentionnally vulnerable. Even if we did not start the scenarios or triggered anything malicious yet, did Defender already catch some issues with this deployment? <br />

Let's jump to security alerts in the same Defender for Cloud portal and filter on our cluster:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155597064-df3dc78b-79dd-4a7c-a1c1-fea0c1ddae57.png" />
</div>
<p></p>

We notice 4 alerts, after the setup:
- Three privileged containers running
- One container with sensitive mount

If we check details of one of these alerts, we can see interesting information, including related [MITRE tactics](https://www.microsoft.com/security/blog/2021/04/29/center-for-threat-informed-defense-teams-up-with-microsoft-partners-to-build-the-attck-for-containers-matrix/), detailed description, pod name...

However we would like to potentially trigger actions based on these alerts or have information about how to mitigate the threat! 
This all happens in the 'take action' tab:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156884308-b0036960-56b6-46b9-93f8-c236d753d06f.png" />
<img src="https://user-images.githubusercontent.com/18376283/155598238-45ee9a3d-c72c-4256-b5c6-8f38cd7f5679.png" />
</div>
<p></p>

1. We can see some detailed advise to mitigate the threats
2. We have a list of ***actionable*** recommendations to prevent this threat in the future

<p></p>
<p></p>

**What are these mitigations exactly?** <br />
Like for preventing vulnerable image in the cluster discussed here above, most of them are leveraging Azure Policy and the admission controller to enforce controls and best-practices on your clusters! 


### But did Defender detect all potential issues and weaknesses?

Only four alerts? Uh. But I thought the goat was damn vulnerable! Good catch! But it is not only about alerts, also about recommendations! In the alert details pane, let's click on 'view all recommendations'...surprise! There is way more problems in our Kubernetes environment than we thought! 

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155598547-8672bb32-2ddf-4e4d-97d5-d7a5e9a7a520.png" />
</div>

<p></p>

We see here recommendations against many best-practices in a Kubernetes environment. These recommendations can easily be mapped to [CIS best-practices](https://www.cisecurity.org/benchmark/kubernetes).
Most of these recommendations can even be remediated or denied in a few clicks:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155598867-45be9135-ab46-4d6b-9716-36e1b65e13c3.png" />
<img src="https://user-images.githubusercontent.com/18376283/155599135-188c465b-361f-4ca2-8de4-779ad3f2c613.png" />
</div>

<p></p>

We can also see a few recommendations are 'met' and compliant with *Healthy* state. You do not have to go or have to wait for an alert to see these best-practice issues. They are listed as recommendations, amongst others in your Defender for Cloud Security Posture Management tab, like for the container runtime CVEs we checked before:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/155599524-362ea59b-dcb8-46c1-bdc4-23b89042dfc8.png" />
</div>

<p></p>

The full list of detection capabilities (alerts) of Defender can be found [here]{https://docs.microsoft.com/en-us/azure/defender-for-cloud/alerts-reference#alerts-k8scluster). <br />
The list of recommendations can be found [here](https://docs.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference)

<a name="Item-7"></a>
## Testing attack patterns

Now, let's play and see how Defender reacts in terms of alerting.<br />
As previously introduced, we will not test all Kubernetes Goat scenarios for various reasons:

1. Some Kubernetes Goat scenarios rely on Docker engine, while AKS use containerd directly (which avoids mandatory root privileges for containers, behind other benefits)
2. Some scenarios are 'inner-network' specific: cross-namespace requests for instance should be tackled with Network Policies such as Calico or Azure CNI. It is not the idea to cover this here, and Defender for Cloud has no way to know what is an authorized applicative flow inside the cluster or not. 
3. Some scenario are related to [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/), however AKS managed clusters are usually exposed by LoadBalancer service rather than NodePort. MDC detects exposure of various sensitive applications using LoadBalancer services.
4. One scenario has for purpose to investigate the inner layers of a container to find crypto mining commands but the job itself is not doing any crypto mining. So while Defender can detect crypto mining activities, it would not be relevant in this context. 
5. Some scenarios would not be spotted: example of the secrets in container images discussed in previous section.
<br />
Details about all the scenarios of the Kubernetes Goat can be found [here](https://madhuakula.com/kubernetes-goat/about.html#:~:text=Kubernetes%20Goat%20is%20designed%20to%20be%20an%20intentionally,production%20environment%20or%20alongside%20any%20sensitive%20cluster%20resources.).
<br />
We will not expand too much on the details here, for the sake of limiting the length of this article, but also because Madhu desceibes already everything you should know in his documentation, and in some good videos such as Defcon talks. The idea is not to spoil solutions but try to capture the flag by yourself if you want to leverage his work!
<br />
Here are the scenarios we will test using Kubernetes Goat setup and see how Defender reacts to that:
- Abusing pod and senstive volume mount: leveraging scenario 2 of Kubernetes Goat, which has remote code execution weakness, to trigger code execution on the pod
- Abusing pod and complete host volume mount: leveraging scenario 4 of Kubernetes Goat, escaping to the host, to triger code execution on the pod and the node this time
- Targeting Kube API: Leveraging scenario 16 and try to play with Kubernetes API and abusing known Kubernetes service accounts threat vectors
- Deploying in the kube-system namespace
- Creating a privileged role

### Abusing pod and senstive volume mount

**Pod used:** Health Check <br />
**Pod properties:** <br />
- Privileged Context  <br />
- Sensitive Mount: In the original version from Kubernetes Goat's author, *health_check* deployment has host docker socket mounted in the pod. This was a recurring threat vector in the Docker world (see, [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)). As discussed, AKS does not leverage Docker daemon anymore, but rather containerd. We therefore replaced the docker.socket mount in this pod, by another sensitive mount, /var from the host and re-deployed the pod:
<p></p>
<div style="text-align: center; max-width: 70%;">
<img src="https://user-images.githubusercontent.com/18376283/157660943-45bb3f4b-88aa-4a5a-ba55-88498528583a.png" />
</div>
<p></p>
As we have seen in the previous section, this raised an alert in Defender, about sensitive volume mount.<br />
Now, we can actually try to abuse the application exposed through this pod. If we navigate to the pod, it is in fact exposing a *health check* service. This service allows to monitor the cluster but it has a catch, it allows for remote code execution on the pod. We can simply execute any command by appending them to the target host to be scanned field. Example, running the `mount` command on the pod:
<p></p>

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156259432-50fdf59e-ced7-4cb1-937d-92ef0bce3fb0.png" />
</div>
                                                                                                                
<p></p>

From there we tested the following commands:

- Downloading a malicious file (*eicar* is a sample malware used for testing purposes) and executing it:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156260030-96be932d-b37a-4dc5-b33e-37167c4ff214.png" />
</div>

<p></p>

We also copied that same eicar file on the host itself, in /var/tmp, using the mounted host volume in /custom/var (which we got from the mount command executed in first step):

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156260343-dbf73620-afc4-42af-a105-d6446ae1f9c6.png" />
</div>

<p></p>

- Deleting backup files in host's /var folder

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156260635-7eab818b-d5bd-4168-9b6d-a84b5c35fd01.png" />
</div>

<p></p>

- Deleting .bash_history to cover our traces: this is a regular evidence hiding step taken by attackers to remove traces of the commands they ran on a system.

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156261238-bbe91fee-a1d2-47ce-b0f8-5f162b300a14.png" />
</div>

<p></p>

### Abusing pod and complete host volume mount

**Pod used:** System Monitor<br />
**Pod properties:**<br />
- Privileged Context <br />
- Sensitive Mount: the full host system / volume is mounted in the pod, in /host-system

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156262001-14395eb9-dc3c-4ec6-a447-da7fe0b7d444.png" />
</div>

<p></p>

In this scenario, we took the following actions:

- Enabled the SSH server on the host to allow for instance for remote access (persistency) and added a newly generated attacker controlled SSH key to authorized keys on the host. We simply used ssh-keygen to generate a new sample key pair for SSH:

`/usr/bin/sshd start~`
`echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCmljeUnkk3eBlavzDmw3Dm5gQJrjCs//3CxJ+4mjbRbja4Hu1o46smTsHJkgXVfwyelUbTTCjRny5LcP9A
4AzQfdLTkhnHrfFNpSja7ouOzL4cyPrgtA0qvJWYPwRvXYAiO9tJ1G6XEYz3Y+i2WbVP+c/OIUUqojldlJaEeFgVu8+TArHUtAx6HOXkpeFE/+6dBEpb/ezG
sGO4Afp9FZxCChVoLLWEa6Bdbdjsq0PxKbp3/+kf9OyJCB2kHRiEyJBDGmXrsgo/cJ5mdgfM0EN2OwKJm9MKH+73LpX2HXH+76Vr5OjiedlKxTfQq37rBpyG
pyE6tQqV1DRLPz7cCi3ALc74FVghQRBVCL/Jv0MXBoBbC3H+Ik148ZyKzGzrBMJkICnaftpQw8fwHkGhzH+zkRK5fwZBS3kEL1lM8hvJ7Q0VxasqOLgpdYAa
sWE4Y1lCLqTdRjtYsuRshcdJj8soa9tKWwwDbiEPANLvuilsyRrwp0YpWwv2XhpjnpWl+gU= " >> ~/.ssh/authorized_keys`

- Escape to the node using chroot (basically changing our root directory to the host's and launching bash from there):

`chroot /host-system bash` 

- Adding a new user to the sudoers group:

`useradd -g sudo Tux`

- Stopping apt-daily-upgrade.timer service

*apt-daily.service* has several functions in Linux: it performs automatic installation of services/packages, it looks for the package updates periodically, it updates the package list daily but also download and install security updates daily.
Stopping this service allows for instance to hide evidence, delay an update or yet download a malicious package and have it run with privileges.
In this case, this could allow us further exploitation on the node (which is self-managed in normal times). 

`systemctl stop apt-daily.timer`

- Let's finally again delete our traces

`rm -rf ~/.bash_history`

### Targeting Kube API

**Pod used:** Hunger Check <br />
**Pod properties:** <br />
- Secret reader RBAC role
<p></p>
This pod has extra-privileges being assigned. A RBAC role 'secret reader' is assigned to the service account bound to this pod. <br />
Service accounts are identities in Kubernetes world, assigned to resources such as pods, which allows them to communicate with the Kube API, with certain privileges. 
A good article on the subject and related threats can be found [here](https://techcommunity.microsoft.com/t5/microsoft-defender-for-cloud/detecting-identity-attacks-in-kubernetes/ba-p/3232340 ). 
<p></p>
In this specific case, the deployment has overly permissive policy/access which allows us to gain access to other resources and services.
<p></p>
<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156607828-fbd4dd88-5d02-4a01-872b-3dab51612b8a.png" />
</div>

<p></p>

As an attacker, we can use the service account token information and try to retrieve information from the Kube API:

`export APISERVER=https://${KUBERNETES_SERVICE_HOST}
export SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
export NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
export TOKEN=$(cat ${SERVICEACCOUNT}/token)
export CACERT=${SERVICEACCOUNT}/ca.crt`

We can then try to list secrets in the default namespace:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156608451-f6b30669-3c32-414f-b555-a28a5fe31533.png" />
</div>

<p></p>

It does not work, as we are not in the default namespace. Indeed, the pod are running in *big-monolith* namespace:

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156608884-3ab21fba-4893-450f-a0bc-a6ca4d7d967a.png" />
</div>

<p></p>

But we can therefore use the correct namespace and run the following command to list secrets:

`curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/namespaces/${NAMESPACE}/secrets`

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156610843-96d8321b-3919-4a00-9bc8-db5eaa6a9a47.png" />
</div>

<p></p>

### Deploying in the kube-system namespace

*Kube-system* namespace is a reserved namespace for all components relating to the Kubernetes system. Deploying in that namespace should be forbidden and can lead to damage or security compromise of the full cluster.

For this scenario, we will use one of the pod provided by Kubernetes Goat, *hacker-container* which is essentially a pod used as hacker playground to further footprinting or compromise a kubernetes/container environment. We will write a simple YAML file to deploy this container in the *kube-system* namespace.

<div style="text-align: center; max-width: 70%;">
<img src="https://user-images.githubusercontent.com/18376283/156728980-c7c3142d-f1c7-46fc-9e7b-c101cbb57646.png" />
</div>

<p></p>

We notice at the same time that this deployment has *NET_ADMIN* capability as well as a sensitive volume mount, */etc*.<br />
Let's deploy that pod in the cluster:

`kubectl apply -f maliciousYAML.yaml`

### Creating a cluster-level privileged role 

Our last exercice for this article will be to create a cluster role. Cluster roles and cluster role bindings are like regular kubernetes (namespaced) roles except they are for a cluster scoped resource. For example a cluster admin role can be created to provide a cluster administrator permissions to vieww or list secrets in the cluster. This can be done using the following *yaml* definition file (giving the role a generic reader name):

`#role.yaml  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
 name: get-pods  
rules:  
  apiGroups: [""] # "" indicates the core API group  
  resources: ["secrets"]  
  verbs: ["get", "watch", "list"]`

We deploy this role into our cluster. We could have created a service account or a user to bind this role to, but we stopped here at cluster role creation.

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156892614-2cefe5b6-9090-493c-8dcf-8fba6c27b656.png" />
</div>

### Looking at the alerts!

Ok so we did a bunch of malicious operations in our environment and it would be good now to check if Defender for Cloud spotted something else than the alerts and recommendations we looked at.<br />
We should of course first, in a real environment, have applied recommendations which we discussed earlier in this article, and enforced policies to avoid malicious operations from happening, but the intent here was to test the alerting capability as well!
<p></p>
If we go back to our Defender for Cloud panel, in the alerts section, we can see we triggered a bunch of alerts:<br />
Most of the scenarios we tested did trigger suspicious activities, next to the ones raised when we deployed Kubernetes Goat in our environment.

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156892691-373635b6-4656-4179-8393-469f85c42887.png" />
</div>

<p></p>

Let's look at a few of them in more details:

#### Detected suspicious use of the useradd command

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156613114-337c5711-19d0-42b6-8d04-bbf4a938265a.png" />
</div>

<p></p>


#### A file was downloaded and executed

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156613357-a6b29344-241d-4fee-9f5c-2fd1a31f13df.png" />
</div>

<p></p>


#### New container in the kube-system namespace detected

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156729443-d5880108-0cf5-4193-a4ab-cb214b01e18f.png" />
</div>

<p></p>

#### Attempt to stop apt-daily-upgrade.timer service detected 

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156892734-18769671-cdc4-4c1e-b423-c038c51da510.png" />
</div>

<p></p>

#### New high privileges role detected

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156892793-f77f19c9-b4e6-4961-954c-760171bb04e3.png" />
</div>

<p></p>


**Note:** <br />
Some tests did not trigger any alerts (deleting backup files in host's /var folder, copying *eicar* file to host from a pod or yet the *NET_ADMIN* capability in our hacker container). <br />
The list of alerts and detection capabilities should grow over time. Defender will however always make a informed decision between raising an alert or in some scenarios relying on a recommendation only (sensitive volume mount in this case), to avoid alert fatigue. <br />
For the capability *NET_ADMIN*, if we look back at [https://docs.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference](recommendations list), it is not one of them. For now, only capability *CAPSYSADMIN* is. You could however easily build/duplicate that policy to look for *NET_ADMIN* as well, hence the power of leveraging Gatekeper and Azure Policies. <br />

<div style="text-align: center">
<img src="https://user-images.githubusercontent.com/18376283/156730083-0480eff8-695f-4f04-8e26-97817ed83603.png" />
</div>

<p></p>

All these recommendations should lead to deploying policies to remediate the security of the environment, and prevent malicious acts leading to alerts from even be possible.

<a name="Item-8"></a>
## Conclusion

In this article we took a vulnerable AKS environment and looked at the capabilities of Defender for Containers including:
- Vulnerability scanning (in registry and at runtime)
- Policies enforcement and/or audit through usage of Azure Policy and Gatekeeper
- Alerting and recommendations from Defender for Cloud on a vulnerable environment where malicious actions were triggered
<p></p>
Important to remember the difference between an alert and a recommendation. If you look only at alerts, you are missing an important part of the security assessment of your cluster, being the continuous evaluation of the seurity posture and preventing the above threats from being possible.  

<a name="Item-9"></a>
## References

[Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)<br />
[Kubernetes Goat](https://madhuakula.com/kubernetes-goat/index.html)<br />
[AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)<br />
[MITRE for Kubernetes](https://www.microsoft.com/security/blog/2020/04/02/attack-matrix-kubernetes/)
[Defender for Cloud Documentation](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction)
<br />
<br />

*Special thanks to Microsoft's Defender for Container Product Group for their support and knowledge sharing*


