---
title: Observing WebSphere Application Server With IBM Instana
date: 2023-01-12 09:00:00 +/-0000
categories: [IBM Instana, websphere]
tags: [aiops, apm, cloudnative]     # TAG names should always be lowercase
image: https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-1-18-Observing-WebSphere-Application-Server-With-IBM-Instana/instanaWAS.png
---
WebSphere Application Server (WAS) has been around for a long time, since 1998 in fact.  Its focus was on running enterprise applications that used the J2EE stack.  Back in those days, applications were monoliths and methodologies were waterfall.  Of course, WebSphere has evolved with the times and the influence of cloud, microservices architectures and agile methodologies spawned the more lightweight application server, WebSphere Liberty, some of which is open sourced via the [Open Liberty project](https://openliberty.io/).  Although WAS is still supported by IBM, it is likely that many customers will be looking to migrate and modernise their applications to make them compatible with Liberty and the modern world!  However, in the interim, this is a perfect example of why an observability solution like Instana is valuable for improving the reliability and resiliency of applications which span heterogeneous environments, including WAS.

Fortunately, Instana is able to collect metrics from WAS and the infrastructure on which it runs.  WAS is one of Instana's built in sensors and data will be automatically collected from the existing [WAS Performance Monitoring Infrastructure](https://www.ibm.com/docs/en/was/9.0.5?topic=health-performance-monitoring-infrastructure-pmi), including:

* Configuration (e.g. cell name, node name, installed apps, datasources etc)
* Performance metrics (e.g. threads, sessions, pool sizes etc)

In addition, Instana provides default Health Signatures and the possibility to define your own.  These are used to automatically trigger events, to help avoid impending issues.  For WAS there is just one health signature which enables you to configure an Alert if the web container threads are reaching the maximum limit.

You can expect Instana to tie together everything from infrastructure, WAS applications and metrics.  Let's take a look at how to set this up, and test a problem scenario.  

## Installing the Instana Agent

Having already installed WAS 9 on an Ubuntu, I also need to install the Instana agent onto the host.  Instana provides several options to achieve this.  I simply ran the script:

![instanaAgentLinux](/assets/img/2023-1-18-Observing-WebSphere-Application-Server-With-IBM-Instana/instanaAgentLinux.png)

Instana provides an Infrastracture view which locates and observe specific hosts.  It uses a query syntax providing entities to query on, e.g. by host name, application, platform etc.  I used a simple query to locate the host where I had installed the Instana agent:

![hostListed1](/assets/img/2023-1-18-Observing-WebSphere-Application-Server-With-IBM-Instana/hostListed1.png)

The host was listed as expected.  However, it showed a zone of "undefined".  When installing the agent on IaaS from some clouds (e.g. AWS), this would be configured automatically.  If not, it's possible to manually set the zone information, which helps to organise the infrastructure monitored by Instana.  The agent is configured by the `/opt/instana/agent/etc/instana/configuration.yaml`, see the [documentation](https://www.ibm.com/docs/en/instana-observability/current?topic=agent-host-configuration#custom-zones) for more detail.  I added the following:
