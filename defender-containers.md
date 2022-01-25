---
layout: default
title: Guillaume's Security Notebook
description: Guillaume B., Cloud Security Architect
---

# Defender for Containers and the goat in the boat: a look at how Defender for Containers protect your clusters

<img src="images/kuby-logo.png" style="float: left;margin: 5px;width: 25%;height: auto;" alt="Defender for Containers" />
(Source: (c) original picture (without the intruder) is from Matt Butcher)

<p style="color:#145DA0;text-align: justify;">Over the last few years, supply chain attacks have been on the rise, and led to evidence that software factories are often not adequately monitored by security teams, or not with the required visibility. <br />
Next to GitHub, GitLab is one of the most commonly used DevOps and source-code repository platform. <br />
Microsoft Sentinel already has some great integration (and more to come...) with GitHub platform (see this <a href="https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/protecting-your-github-assets-with-azure-sentinel/ba-p/1457721">excellent article</a>) but no existing connector for GitLab as of the time of writing. <br />
Inspired by the article on GitHub and Sentinel and following discussions with some customers using GitLab themselves, I wrote parsers and analytics rules for Gitlab environment, in order to give SOC the required visibility on threats related to GitLab environment.<br />
These queries are just a few and much more could be build out of GitLab logs, feel free to adapt or extend based on your own needs.</p>
