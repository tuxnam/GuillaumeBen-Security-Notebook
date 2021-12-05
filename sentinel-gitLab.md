---
layout: default
title: Guillaume's Security Notebook
description: Guillaume Benats, Cloud Security Architect @ Microsoft
---

# Monitoring threats to your GitLab environment using Microsoft Sentinel

<span style="color:#145DA0;">*In the recent years, supply chain attacks have been on the rise, and software factories are often not adwqutely monitored by security teams, or not with the required focus. Next to GitHub, GitLab is one of the most commonly used source-code repository platform. Microsoft Sentinel already has some great integration (and more to come...) with GitHub platform (see this <a href="https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/protecting-your-github-assets-with-azure-sentinel/ba-p/1457721">excellent article</a>) but no existing connector for GitLab as of the time of writing.
Inspired by the article here above and after discussions with some customers using GitLab self-hosted, my goal is to provide parsers and analytics rules for Gitlab environment, in order to give SOC the required visibility on threats related to your supply chain.*</span>

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

### Parsers

The first thing to do when GitLab logging data starts to be ingested into Sentinel, is to parse the SyslogMessage accordingly.
I created three parsers according to the logging sources:
- One parser for GitLab NGINX access logs
- One parser for GitLab audit logs
- One parser for GitLab application logs

The goal of these parsers is to register them as *functions* (see [here](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-functions-template?tabs=ARM)) in your Sentinel environment to then use them seamlessly as data sources in your hunting queries without having to parse again each time manually. 

##### GitLab Audit Logs parser

```yaml
// GitLab Enterprise Edition Audit Logs Data Parser
// Last Updated Date: Nov 30, 2021
//
// This parser parses GitLab standalone Enterprise Edition audit logs extract the infromation required for all audit queries on GitLab. 
// It is assumed that the syslog data connector to ingest GitLab audit entry data into Sentinel is enabled
//
// Parser Notes:
// 1. This parser assumes logs are collected into native Syslog table, including audit_json logs
// 2. This parser assuments that GitLab syslog configuration is leveraging 'ProcessName' and 'Facility' to categorize syslog events
//    Example: ProcessName for audit logs would be (adapt according to your own configuration) "GitLab-Audit-Logs" and the Facility is "local7"
//
// Usage Instruction : 
// Paste below query in log analytics, click on Save button and select as Function from drop down by specifying function name and alias. 
// To work with analytics rules built next to this parser, this Function should be given the alias of GitLabAudit.
// Functions usually takes 10-15 minutes to activate. You can then use function alias from any other queries (e.g. GitLabAudit | take 10).
//
// References : 
// Using functions in Azure monitor log queries : https://docs.microsoft.com/azure/azure-monitor/log-query/functions
// Tech Community Blog on KQL Functions : https://techcommunity.microsoft.com/t5/Azure-Sentinel/Using-KQL-functions-to-speed-up-analysis-in-Azure-Sentinel/ba-p/712381
// Understanding GitLab logging: https://docs.gitlab.com/ee/administration/logs.html
//

Syslog
| where Facility == 'local7'
| where ProcessName contains 'audit'
| extend parsedMessage = parse_json(SyslogMessage)
| project TimeGenerated, 
  Severity = parsedMessage.severity,
  EventTime = parsedMessage.['time'],
  CorrelationID = parsedMessage.correlation_id,
  AuthorID = parsedMessage.author_id,
  AuthorName = parsedMessage.author_name,
  EntityID = parsedMessage.entity_id,
  EntityType = parsedMessage.entity_type,
  IPAddress = parsedMessage.ip_address,
  AddAction = parsedMessage.['add'],
  DelAction = parsedMessage.['del'],
  RemoveAction = parsedMessage.['remove'],
  TargetID = parsedMessage.target_id,
  TargetType = parsedMessage.target_type,
  TargetDetails = parsedMessage.target_details,
  EntityPath = parsedMessage.entity_path,
  Action = parsedMessage.action,
  CustomMessage = parsedMessage.custom_message,
  AuthenticationType = parsedMessage.['with'],
  SourceVisibility = parsedMessage.From,
  TargetVisibility = parsedMessage.To,
  ChangeType = parsedMessage.change
```

##### GitLab Application parser

##### GitLab NGINX access parser
