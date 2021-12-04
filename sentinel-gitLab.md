---
layout: default
title: Guillaume's Security Notebook
description: Guillaume Benats, Cloud Security Architect @ Microsoft
---

# Monitoring threats to your GitLab environment using Microsoft Sentinel

> Next to GitHub, GitLab is one of the most commonly used source-code repository platform. 
> In the recent years, supply chain attacks have been on the news, and software factories are often not adwqutely monitored
> by security teams, or not with the required focus. 
> Microsoft Sentinel already has some great integration (and more to come...) with GitHub platform (see this <a href="https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/protecting-your-github-assets-with-azure-sentinel/ba-p/1457721">excellent article</a>) but no existing connector for GitLab as of time of writing.
> Inspired by the article here above, my goal is to provide parsers and analytics rules for Gitlab standalone environment, Enterprise or Premium edition. The goal is to give SOC the required visibility on actions taken in your supply chain.
