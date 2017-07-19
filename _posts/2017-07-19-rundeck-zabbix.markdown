---
title: "Runbook Automation with Zabbix and Rundeck"
layout: post
date: 2017-07-19 12:29
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rundeck
- zabbix
- automation
category: blog
author: kosyanyanwu
description: Integrate Zabbix trigger based actions into Rundeck
---

## Introduction

[Zabbix](https://www.zabbix.com/) is an open source enterprise-level software designed for real-time monitoring of millions of metrics.

[Rundeck](http://rundeck.org/) is a job scheduler and runbook automation tool.

This article explains how you can integrate Zabbix trigger based actions into Rundeck.

## Real world example
* A service stops running
* Zabbix fires trigger and sends notifications
* Ops receives notification
* Ops goes to restart service
* Hurray!!! service is running again
* Probably another one stops running in another server
* And the cycle continues ...

## Disadvantages
* Process is repetitive, manual and sometimes annoying :(
* No easily accessible preserved output from executed remote actions

## Zabbix meets Rundeck (Real world example)
* A service stops running
* Zabbix fires trigger
* Zabbix action calls middleware
* Middleware executes job on Rundeck
* Middleware sends acknowledgement to Zabbix
* Zabbix receives acknowledgement
* Ops continues sipping coffee, nothing left for them to do

## Integration requirements
* Map Zabbix hosts to Rundeck nodes
* Map Zabbix trigger names to Rundeck job names
* On trigger event, pass related host and trigger from Zabbix to Rundeck
* Return job execution status from Rundeck as an event acknowledgment in Zabbix
