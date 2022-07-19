layout: post
title: Guillaume's Security Notebook
description: Guillaume B., Security Analyst (@ Microsoft)
---

#### Last update: July 2022

# Collecting and correlating security events in AD using Microsoft Sentinel

<p></p>
<span class="subtitle">In this article, we will explore how to collect interesting events from an Active Directory Domain Controller host into Microsoft Sentinel, and how these specific events can be mapped to MITRE tactics and correlate them into valuable incidents. </span>

## Important notes for this article

- We solely focus on events (security events, windows events, sysmon...) and not on any EDR solution which could (should) be deployed in most real-life scenarios
- Even if an EDR, such as Microsoft Defender for Endpoint, is available on the Domain Controller, *raw* events are always useful to give additional context or cover specific scenarios in a SIEM solution, specifically for domain controllers
- Specific products for AD security, such as Microsoft Defender for Identity are out-of-scope of this article
- The article is not exhaustive but rather focuses on collecting events to gain information about well-known TTPs used by attackers these days
<p></p>

## Take the MITRE ATT&CK framework as a baseline

### List of relevat techniques for a Domain Controller

## Understand and change default Windows security audit settings

## Sysmon or not sysmon?

## Configuring required events 

## Collecting events into Microsoft Sentinel

## Working with the data
