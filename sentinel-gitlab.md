---
layout: default
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

# Monitoring threats to your GitLab environment using Microsoft Sentinel

<img src="images/GitLab-Sentinel.png" style="float: left;margin: 5px;width: 25%;height: auto;" alt="Monitor GitLab using Sentinel" />
<p style="color:#145DA0;text-align: justify;">Over the last few, supply chain attacks have been on the rise, and led to evidence that software factories are often not adequately monitored by security teams, or not with the required visibility. <br />
Next to GitHub, GitLab is one of the most commonly used DevOps and source-code repository platform. <br />
Microsoft Sentinel already has some great integration (and more to come...) with GitHub platform (see this <a href="https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/protecting-your-github-assets-with-azure-sentinel/ba-p/1457721">excellent article</a>) but no existing connector for GitLab as of the time of writing. <br />
Inspired by the article on GitHub and Sentinel and following discussions with some customers using GitLab themselves, I wrote parsers and analytics rules for Gitlab environment, in order to give SOC the required visibility on threats related to GitLab environment.<br />
These queries are just a few and much more could be build out of GitLab logs, feel free to adapt or extend based on your own needs.</p>

<a href="https://github.com/tuxnam/Sentinel-Development" target="_blank" style="margin-top: 1em; margin-bottom: 1em; display: inline-flex;  text-decoration: none; color: black;"><img src="images/GitHub-Icon.png" style="width: 28px; height: auto; vertical-align: middle;" />&nbsp;&nbsp;Check sources on my GitHub repo!</a>

## GitLab versions and required logs

GitLab has two main SKUs: a free, community edition and an enterprise edition. See details [here](https://about.gitlab.com/install/ce-or-ee/). <br />
This article focuses on GitLab self-hosted Enterprise Edition (EE). Audit events are not available in the Community Edition (CE) but some of the hunting queries I built require only application logs or server access logs and should therefore also work for CE edition. 

All the information regarding the logging system in GitLab can be found in the [official documentation](https://docs.gitlab.com/ee/administration/logs.html).

The logs used in the scope of this Microsoft Sentinel integration are:
- GitLab Audit logs: changes to group or project settings and memberships (target_details) are logged to this file.
- GitLab NGINX Access logs: a log of requests made to GitLab.
- GitLab Application logs: helps you discover events happening in your instance such as user creation and project deletion.

There are much more log files part of GitLab but most of the interesting security events will be captured in these three files. 

## Ingesting GitLab logs to Sentinel

The first step is to actually ingest GitLab data into Microsoft Sentinel. <br />
GitLab is using syslog for logging and we will therefore use the syslog connector in Microsoft Sentinel. <br />
GitLab logs are written (by default) into */var/log/gitlab*. <br />
In order to ingest syslog data into Microsoft Sentinel, we will need to deploy the log analytics agent on the host, and connect it to the Sentinel workspace where we want to query the data. The simplest way to install the agent is throug [VM extensions](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/oms-linux) but you can also scale it through [scripts ormanual deployments](https://docs.microsoft.com/en-us/azure/azure-monitor/agents/agent-linux). <br /> 
Once this is done, we just need to instruct the log analytics agent now installed on the host to collect the specific facility which we will use for GitLab syslog messages on the system (local7 in our case)(see the [official Documentation](https://docs.microsoft.com/en-us/azure/sentinel/connect-syslog#configure-the-log-analytics-agent)). <br />
This also means, once the agent is installed on the host, that we need to define syslog configuration files for GitLab audit, application and NGINX access logs, in regular syslog configuration folder (*/etc/rsyslod.d/* for rsyslog), along with the syslog facility we configured here above (i.e: local1, local2, local7, syslog...) (samples of syslog configuration files can be found in the [corresponding GitHub repository](https://github.com/tuxnam/Sentinel-Development/tree/main/Syslog/GitLab)). <br /> 
The agent will in fact create a collection configuration file in the same syslog  (/etc/rsyslog.d/) repository, with instructions to ingest logs of the selected facility.
<br />
<br />
*Example:*
~~~
# /etc/rsyslog.d/95-omsagent.conf 
# OMS Syslog collection for workspace 6a02aa7b-ed4b-4f13-9220-ed080c0da1f5
local7.=alert;local7.=crit;local7.=debug;local7.=emerg;local7.=err;local7.=info;local7.=notice;local7.=warning@127.0.0.1:25224
~~~
<br />
We should now start to see logs being ingest into *Syslog* table in Sentinel. <br />

This article does not intend to deep-dive into syslog configuration or logs ingestion into Sentinel, all details about syslog collection [here](https://docs.microsoft.com/en-us/azure/sentinel/connect-syslog).<br /><br />
**Note:** the advantage of using syslog is that logs are ingested directly into Sentinel *Syslog* native table. You could also ingest GitLab logs using Sentinel API and a custom table in the log analytics workspace behind Sentinel (*GitLab_Audit_logs_CL* for instance) but there are drawbacks compared to using native tables (correlation in fusion rules is one of them). The goal is not to cover this in this article, please refer to [official documentation](https://docs.microsoft.com/en-us/azure/sentinel/connect-data-sources).

**Note 2:** usually, in a corporate environment, you would use a syslog server and forward GitLab logs to it. The principle is the same, except that the ingestion should then happen from the syslog server and not GitLab host. 

## Parsers

The first thing to do when GitLab logging data starts to be ingested into Sentinel, is to parse the syslog messages accordingly.
I created three parsers to ease the writing of hunting queries, based on the logging sources:
- [One parser for GitLab NGINX access logs](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_Access)
- [One parser for GitLab audit logs](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_AuditLogs)
- [One parser for GitLab application logs](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_AppLog)

The goal of these parsers is to be registered as *functions* (see [here](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-functions-template?tabs=ARM)) in our Sentinel environment to then use them seamlessly as data sources in hunting queries without having to parse again each time manually. 

In a few simple steps:<br />
We have to paste below parser queries in log analytics, click on <u>save</u> button and select *'as Function'* from drop down by specifying function name and alias. <br />
To work with analytics rules built next to this parser, these functions should be given the alias of *GitLabAudit*, *GitLabApplication* and *GitLabAccess* respectively.<br />
Functions usually takes a few minutes to activate. We can then use function alias from any other queries (e.g. *GitLabAudit | take 10*).<br />
Feel free to rename them and adapt the name in the hunting queries below in this article.

**Note:** In my case I used syslog Facility *local7* and *ProcessName* in ('GitLab-Audit-Logs', 'GitLab-Application-Logs', 'GitLab-Access-Logs') in the rsyslog.d configuration files for audit, application and NGINX access logs respectively. Feel free to use your own Facility or ProcessName and adapt the below parsers.

#### GitLab Audit Logs parser ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_AuditLogs))

**Description:** This parser parses GitLab audit logs (*audit_json.log*) to extract the infromation required for all audit queries on GitLab. 

```
// GitLab Enterprise Edition Audit Logs Data Parser
// Last Updated Date: Dec 3, 2021
//

Syslog
| where Facility == 'local7'
| where ProcessName contains 'GitLab-Audit-Logs'
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

#### GitLab Application parser ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_AppLog))

**Description:** This parser parses GitLab application logs (*application.log*) to extract the infromation required for all application events queries on GitLab. 

```
// GitLab Enterprise Edition Application Logs Data Parser
// Last Updated Date: Dec 3, 2021
//

Syslog
| where Facility == 'local7'
| where ProcessName contains 'GitLab-Application-Logs'
| project TimeGenerated, Computer, HostName, HostIP, Message = SyslogMessage
```

#### GitLab NGINX access parser ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/Parsers/GitLab/GitLab_Access))

**Description:** This parser parses GitLab NGINX access logs (*access.log*) to extract the infromation required for all requests landing on GitLab server. 

```
// GitLab Enterprise Edition Application Logs Data Parser
// Last Updated Date: Dec 3, 2021
//

Syslog
| where Facility == 'local7'
| where ProcessName == 'GitLab-Access-Logs'
| where SyslogMessage has_any ("GET","POST","PUT","DELETE","PATCH") and SyslogMessage contains "HTTP"
| parse SyslogMessage with IPAddress " - - [" EventTime "] \"" RequestVerb " " URI " " HTTPVersion "\"" ResponseCode " " BytesSent "\"" HTTPReferer "\" \"" UserAgent
| project TimeGenerated, EventTime, IPAddress, RequestVerb, URI, HTTPVersion, ResponseCode, BytesSent, HTTPReferer, UserAgent
```

## Hunting Queries

And now into the real 'meat' with the actual queries to be used for analytics rules or threat hunting for GitLab in Microsoft Sentinel.

#### Identify brute-force attempts on GitLab ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_BruteForce))

**Description:** This query relies on GitLab Application Logs to get failed logins to highlight brute-force attempts from different IP addresses in a short space of time.<br />
**Parameters:** learning period, time window, thresholds

~~~
let LearningPeriod = 7d; 
let BinTime = 1h; 
let RunTime = 1h; 
let StartTime = 1h; 
let NumberOfStds = 3; 
let MinThreshold = 10.0; 
let EndRunTime = StartTime - RunTime; 
let EndLearningTime = StartTime + LearningPeriod;
let GitLabFailedLogins = (GitLabApp
| where Message contains "Failed Login"
| parse kind=regex Message with EventTime ": Failed Login: username=" Username "ip=" IpAddress 
| project TimeGenerated, EventTime, Username, IpAddress);
GitLabFailedLogins 
  | where todatetime(EventTime) between (ago(EndLearningTime) .. ago(StartTime)) 
  | summarize FailedLoginsCountInBinTime = count() by User = Username, bin(todatetime(EventTime), BinTime) 
  | summarize AvgOfFailedLoginsInLearning = avg(FailedLoginsCountInBinTime), StdOfFailedLoginsInLearning = stdev(FailedLoginsCountInBinTime) by User 
  | extend LearningThreshold = max_of(AvgOfFailedLoginsInLearning + StdOfFailedLoginsInLearning * NumberOfStds, MinThreshold) 
  | join kind=innerunique ( 
    GitLabFailedLogins 
    | where todatetime(EventTime) between (ago(StartTime) .. ago(EndRunTime)) 
    | summarize FailedLoginsCountInRunTime = count() by User = Username, IpAddress 
  ) on User 
  | where FailedLoginsCountInRunTime > LearningThreshold
  | extend User, IpAddress
~~~

#### External user added on GitLab ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_ExternalUser))

**Description:** This query relies on GitLab Application logs to list external user accounts (i.e.: account not in allow-listed domains) which have been added to GitLab users.<br />
**Parameters:** Allow-list of domains.

~~~
// List of allow-listed domains
let allowedDomain = pack_array("mydomain.com");
GitLabAudit
| where AddAction == "user"
| project AuthorName, IPAddress, UserAdded = Entity_path
| join (GitLabApp 
| where Message contains "User" and Message contains " was created" 
| parse kind=regex Message with EventTime ": User \"" Username "\"" EmailAddress " was created"
| project UserAdded = tostring(Username), EmailAddress = substring(EmailAddress,2,strlen(EmailAddress)-3)) on UserAdded
| project  AuthorName, IPAddress, UserAdded, EmailAddress, DomainName = tostring(extract("@(.*)$", 1, EmailAddress))
| where allowedDomain !contains DomainName
~~~


#### Actions done under user impersonation ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_Impersonation))

**Description:** This query relies on GitLab Audit Logs for user impersonation. A malicious operator or a compromised admin account could leverage the impersonation feature of GitLab to change code or repository settings bypassing usual processes. This hunting queries allows you to track the audit actions done under impersonation. <br />
**Parameters:** /

~~~
let impersonationStart = (GitLabAudit
| where CustomMessage == 'Started Impersonation'
| extend AuthorID = tostring(AuthorID), TargetID = tostring(AuthorID));
let impersonationStop = (GitLabAudit
| where CustomMessage == 'Stopped Impersonation'
| extend AuthorID = tostring(AuthorID), TargetID = tostring(AuthorID));
impersonationStart
| join kind=inner impersonationStop on $left.TargetID == $right.TargetID
and $left.AuthorID == $right.AuthorID 
| where todatetime(EventTime1) > todatetime(EventTime)
| extend AuthorID, AuthorName, TargetID, TargetDetails = tostring(TargetDetails), IPStart = IPAddress, IPStop = IPAddress1, ImpStartTime = EventTime, ImpStopTime = EventTime1, EntityPath
| join kind=inner (GitLabAudit | extend ActionTime = EventTime, AuthorName = tostring(AuthorName)) on $left.TargetDetails == $right.AuthorName 
| where todatetime(ImpStartTime) < todatetime(ActionTime) and todatetime(ActionTime) > todatetime(ImpStopTime)
~~~

#### Local authentication without multi-factor authentication ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_LocalAuthNoMFA))

**Description:** This query checks GitLab Audit Logs to see if a user authenticated without MFA. Ot might mean that MFA was disabled for the GitLab server or that an external authentication provider was bypassed. This rule focuses on 'admin' privileges but the parameter can be adapted to also include all users.<br />
**Parameters:" admin or not

~~~
let isAdmin = true;
GitLabAudit
| where AuthenticationType == "standard" and ((isAdmin and TargetDetails contains "Administrator") or (isAdmin==false));
~~~

#### Threat Intelligence flagged IP accessing GitLab ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_MaliciousIP.yaml))

**Description:** This query correlates Threat Intelligence data from Sentinel with GitLab NGINX Access Logs (available in GitLab CE as well) to identify access from potentially TI-flagged IPs.<br />
**Parameters:** /

~~~
ThreatIntelligenceIndicator
  | where Action == true
  // Picking up only IOCs that contain the entities we want
  | where isnotempty(NetworkIP) or isnotempty(EmailSourceIpAddress) or isnotempty(NetworkDestinationIP) or isnotempty(NetworkSourceIP)
  // Taking the first non-empty value based on potential IOC match availability
  | extend TI_ipEntity = iff(isnotempty(NetworkIP), NetworkIP, NetworkDestinationIP)
  | extend TI_ipEntity = iff(isempty(TI_ipEntity) and isnotempty(NetworkSourceIP), NetworkSourceIP, TI_ipEntity)
  | extend TI_ipEntity = iff(isempty(TI_ipEntity) and isnotempty(EmailSourceIpAddress), EmailSourceIpAddress, TI_ipEntity)
  | join (
GitLabAccess) on $left.TI_ipEntity == $right.IPAddress
 | summarize LatestIndicatorTime = arg_max(TimeGenerated, *) by IndicatorId
 | project LatestIndicatorTime, Description, ActivityGroupNames, IndicatorId, ThreatType, Url, ExpirationDateTime, ConfidenceScore, TimeGenerated = EventTime, TI_ipEntity, IPAddress, URI
 ~~~
 
#### Personal Access Tokens creation over time ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_PAT_Repo))
 
**Description:** This query uses GitLab Audit Logs for access tokens. Attacker can exfiltrate data from you GitLab repository after gaining access to it by generating or hijacking access tokens. This hunting queries allows you to track the personal access tokens creation or each of your repositories. The visualization allow you to quickly identify anomalies/excessive creation, to further investigate repo access & permissions.<br />
**Parameters:** minimum tokens created per day to consider, date to start

```
// l_min_tokens_created - minimum tokens created per repository per day to consider
let l_min_tokens_created = 1;
// Earliest date from GitLab Audit logs - adapt if too far away - for instance, replace by ago(30d)
let min_t = toscalar(GitLabAudit
| summarize min(TimeGenerated));
// Most recent date from GitLab Audit logs 
let max_t = toscalar(GitLabAudit
| summarize max(TimeGenerated));
// Graph Interval
let interval = 1d;
GitLabAudit
| where TargetType == "PersonalAccessToken"
| project Severity, EventDay = bin(todatetime(EventTime),1d), AuthorName, IPAddress, Repository = tostring(EntityPath), Action, TargetType
| summarize sumTokens = count() by Repository, EventDay
| where sumTokens > l_min_tokens_created
| make-series num=sum(sumTokens) default=0 on EventDay in range(ago(30d), now(), interval) by Repository
| extend (anomalies, score, baseline) = series_decompose_anomalies(num, 1.5, -1, 'linefit')
| render timechart
```

#### Repository visbility changed to Public ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_RepoVisibilityChange))

**Description:** This query leverages GitLab Audit Logs. A repository in GitLab changed visibility from Private or Internal to Public which could indicate compromise, error or misconfiguration leading to exposing the repository to the public.<br />
**Parameters:** /

~~~
GitLabAudit
| where SourceVisibility == "Public" and ChangeType == "visibility" and TargetVisibility != "Public"
| project EventTime, IPAddress, AuthorName, ChangeType, TargetType, SourceVisibility,  TargetVisibility, EntityPath
~~~

#### Unusual number of repositories deleted ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_Repo_Deletion))

**Description:** This hunting query identify an unusual increase of repo deletion activities adversaries may want to disrupt availability or compromise integrity by deleting business data.
**Parameters:** learning period, time window, thresholds

~~~
 let LearningPeriod = 7d;
 let BinTime = 1h;
 let RunTime = 1h;
 let StartTime = 1h;
 let NumberOfStds = 3;
 let MinThreshold = 10.0;
 let EndRunTime = StartTime - RunTime;
 let EndLearningTime = StartTime + LearningPeriod;
 let GitLabRepositoryDestroyEvents = (GitLabAudit
 | where RemoveAction == "project" or RemoveAction == "repository");
 GitLabRepositoryDestroyEvents
 | where TimeGenerated between (ago(EndLearningTime) .. ago(StartTime))
 | summarize count() by bin(TimeGenerated, BinTime)
 | summarize AvgInLearning = avg(count_), StdInLearning = stdev(count_)
 | extend LearningThreshold = max_of(AvgInLearning + StdInLearning * NumberOfStds, MinThreshold)
 | extend Dummy = 1
 | join kind=innerunique (
   GitLabRepositoryDestroyEvents
   | where TimeGenerated between (ago(StartTime) .. ago(EndRunTime))
   | summarize CountInRunTime = count() by bin(TimeGenerated, BinTime)
   | extend Dummy = 1
 ) on Dummy
 | project-away Dummy
 | where CountInRunTime > LearningThreshold
~~~

#### Sign-in Bursts ([GitHub](https://github.com/tuxnam/Sentinel-Development/blob/58011386c48e3d02b9f744fe5f60495843d3f42f/AnalyticsRules/GitLab/GitLab_SignInBurst))

**Description:** This query relies on Azure Active Directory sign-in activity when Azure AD is used for SSO with GitLab to highlights GitLab accounts associated with multiple authentications from different geographical locations in a short space of time.<br />
**Parameters:** minimum number of different locations, GitLab application name in AAD

~~~
let locationCountMin = 1;
let appRegistrationName = "GitLab";
SigninLogs
| where AppDisplayName == appRegistrationName
| where ResultType == 0
| where Location != ""
| summarize CountOfLocations = dcount(Location), Locations = make_set(Location) by User = Identity
| where CountOfLocations > locationCountMin
~~~
