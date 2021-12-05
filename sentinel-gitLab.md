---
layout: default
title: Guillaume's Security Notebook
description: Guillaume Benats, Cloud Security Architect @ Microsoft
---

# Monitoring threats to your GitLab environment using Microsoft Sentinel

<span style="color:#145DA0;">***In the recent years, supply chain attacks have been on the rise, and software factories are often not adwqutely monitored by security teams, or not with the required focus. Next to GitHub, GitLab is one of the most commonly used source-code repository platform. Microsoft Sentinel already has some great integration (and more to come...) with GitHub platform (see this <a href="https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/protecting-your-github-assets-with-azure-sentinel/ba-p/1457721">excellent article</a>) but no existing connector for GitLab as of the time of writing.
Inspired by the article here above and after discussions with some customers using GitLab self-hosted, my goal is to provide parsers and analytics rules for Gitlab environment, in order to give SOC the required visibility on threats related to your supply chain.***</span>

### GitLab versions and required logs

This articles focuses on GitLab self-hosted Enterprise Edition (EE)(or higher SKU like Premium). Audit events are not available in the Community Edition (CE) but some of these analytics rules require only application logs or NGINX access logs and will therefore also work for CE edition. 

More details on logging system in GitLab: https://docs.gitlab.com/ee/administration/logs.html

The logs used in the scope of the Microsoft Sentinel integration are:
- GitLab Audit logs: changes to group or project settings and memberships (target_details) are logged to this file.
- GitLab NGINX Access logs: a log of requests made to GitLab.
- GitLab Application logs: helps you discover events happening in your instance such as user creation and project deletion.

There are much more log files part of GitLab but most of the interesting security events will be captured in these three categories. 

### Connecting GitLab server and ingesting logs to Sentinel

The first step is to actually ingest GitLab data into Microsoft Sentinel. We are using here the standard syslog output. GitLab logs are written into */var/log/gitlab*. 
In order to ingest syslog data into Microsoft Sentinel, you will need to deploy the log analytics agent on the host, and connect it to the Sentinel workspace. 
On the host itself, you need to define syslog configuration files for GitLab audit, application and NGINX access logs, in regular /etc/rsyslod.d/, along with the syslog facility you want to use (i.e: local1, local2, local7, syslog...) (samples can be found in the [corresponding GitHub repository](test)). Once this is done, you just need to instruct the agent to collect the specific facility of syslog messages on your system (local7 in our case). <br />
The agent will in fact create a collection configuration file in the sane /etc/rsyslog.d/ repository, to instruct the agent to ingest logs of the selected facility.
All details about syslog collection [here](https://docs.microsoft.com/en-us/azure/sentinel/connect-syslog).

**Note:** the advantage of using syslog is that logs are ingested directly into Sentinel Syslog native table. You could also ingest GitLab logs using a custom table in the log analytics workspace behind Sentinel (*GitLab_Audit_logs_CL* for instance) but there are drawbacks conpared to using native tables (correlation and machine learning is one of them). The goal is not to cover this in this article, please refer to [official documentation](https://docs.microsoft.com/en-us/azure/sentinel/connect-data-sources).
