---
layout: post
title:  "Evaluation of Microsoft Azure Cloud for Java/JEE applications"
date:   2016-06-06 10:05:48 -0500
categories: blog
description: Java, JEE, JBoss Wildfly, Microsoft Azure, Azure App Service, Azure Container Service, Nomad, Apache Mesos 
comments: true
---
The purpose of this article is to describe the outcome of evaluation of Azure Cloud services for deployment of a JEE application.

## 1. Application based on JEE Standards running in Wildfly 10

The Wildfly 10 JEE container makes use of a bundled open source caching solution, Infinispan from Red Hat, which maintains session and level 2 (L2) cache for JPA. In order to achieve failover, these session data needs to be replicated across each node or distributed outside in an in-memory caching store like Redis cache.

### 1.1 Session Replication
The session replication in Wildfly is achieved via a bundled open source solution, JGroups. The JGroups uses TCP/UDP socket level abstraction to form a cluster of nodes that advertise their IP address. The Infinispan works with JGroups to replicate session data across cluster. 

The JGroups solution requires opening a firewall for inbound and outbound TCP/UDP traffic to each node on a configurable port (default 7600).

### 1.2 Distributed Cache
The Infinispan cache can push the data to external store like Redis cache or to Red Hat Data Grid built upon Infinispan. Unfortunately these latest features of Infinispan are not integrated in Wildfly yet.

## 2. Azure Cloud

The Azure cloud provides various deployment options that is claimed to work for a variety of programming runtime. This article describes
where Azure works and where it limits capabilities for Java/JEE based services.

### 2.1 Classic vs. ARM

The Azure cloud has gone through a significant transformation and old version is simply known as Azure Classic. The new version is loosely called Azure Resource Manager or V2. Most of the documentation available are targeted to classic version and newer documentations are coming up but there is not whole lot of information available for troubleshooting Java based deployment in V2.

### 2.2 Azure Resource Manager (ARM) Template

Azure cloud allows us to store cloud resources (VM, Load Balancer, Virtual Network, Network Security Groups etc.) configuration as code using Azure Resource Manager (ARM) templates. These templates can be deployed using various tools – Azure PowerShell, Azure CLI, Visual Studio, Azure Web Interface and Azure REST API etc. All these tools ultimately use the same set of Azure REST API.

Although, Azure PowerShell works the best for scripting the cloud environment, it requires Windows OS which may not be practical for all the Java/JEE developers who are more accustomed to Linux based OS. The Azure CLI which works across OS can be made to work for 95% cases. The cases where Azure CLI was found lacking, as compared to PowerShell counterpart, are when we need to retrieve 2nd level resource information (e.g. publishing credentials of the site created) as output of ARM template execution.

The ARM template is esoteric. The MS Visual Studio provides intelli-sense support to author ARM template but requires Windows OS. Without VS, it is kind of hard to author an ARM template but it gets better over a period of time and snippets can be found from https://resoureces.azure.com for manually created resources.

### 2.3 Azure App Service

The Azure App Service is a server-less deployment model. It lets us focus on code and takes care of the complexity of choosing runtime environment, patching OS, load balancing, providing HA, elastic scaling and monitoring.

The deployment of Azure App Service requires publishing of IIS web.config file and binaries to be executed as configured in http handler section of web.config. The publishing methods are FTP, Git and TFS. We can use git or FTP publishing method to upload Wildfly 10 JEE container with bundled application WAR or subsystems.

The Azure App Service also supports multiple slots for blue/green deployment model. These slots can also be used to distinguish environments like Dev, QA, Production etc.

With all the goodies that Azure App Service provides, it limits Wildfly implement HA/Failover because it is designed for a stateless web application. The Azure App Service executes http based process (e.g. Wildfly container) fronted by IIS proxy on a Windows Server in a shared environment. The Azure configures app service with 4 outbound IP addresses that it can choose from. The process itself is very isolated and does not know its internal IP address. It sees only a loopback address 127.0.0.1.

However, for JGroups cluster, each member must advertise its IP address and port that can be reached later from outside. Azure App Service does not even allow firewall configuration to open additional ports for ingress.

The Azure App Service Environment (ASE) is another option to create a dedicated virtual network (VNet) for the app service but it is not available for Azure V2. The Azure App Service allows setting up connectivity to a VNet for accessing backend services but this connectivity does allow VNet service to reach out to App Service on a well known IP address and port.

### 2.4 Azure Cloud Service

The Azure Cloud Service is similar to Azure App Service but provide access to the Windows VM running the code. The deployment model requires packaging of application as Azure Cloud Package using cloud definition and cloud package configuration files. Building such a cloud package requires MSBuild or Ant build script and Azure SDK. This option was not tried out but appears to suffer from similar limitations.

### 2.5 Azure Container Service

The Azure Container Service (ACS) is a Docker container management service. It uses open source solution, Apache Mesosphere with options of Apache Marathon or Docker Swarm scheduler. The ACS requires running multiple VMs – a set for Master nodes and two sets for Slave nodes (public and private) with a Zookeeper quorum to elect master. The master nodes are lightweight management nodes and slave nodes are the real workhorses, i.e. VMs running Docker containers.

The ACS is in very early stage and suffers from following limitations.

1.	Support for private Docker repository

    Once we schedule launch of a Docker container from Marathon scheduler, the slave node chosen for the job needs a way to pull container from the private repository. But a private repository needs authentication. This requires encrypted Docker authentication token to be uploaded to each Slave node. Since number of slaves are dynamic and we don’t have access to them from scheduler, the slaves fail to launch container.

    Work Around:

    Using CURL, query the master node to get a list of available slaves and their IP addresses. Then upload Docker encrypted authentication key to each slave node. But there is no SSH access to slave nodes. 

    Next Work Around:

    Although, each node including slaves have your SSH public key as authorized keys but SSH connectivity to slave nodes are allowed only from the master node. This requires uploading your SSH private key to the master node so that you can SSH to slaves from the master node.

2.	Elastic scaling of VMs (Mesos Slaves)

    The Azure configures Slave nodes on Azure VM Scale Set which means elastically Azure will add/remove VMs as determined from scaling parameters configuration. However, if we start with 2 VMs and both of them are fully utilized and running healthy then there is nothing to tell Azure that you need another VM to launch few more Docker containers. 

    This docker scheduler and VM scaling integration is missing and must be implemented outside of Azure using APIs and/or external monitoring means to scale out/in the number of VMs needed to maintain a healthy count of Docker containers.

3.	Load Balancer

    If we treat one ACS cluster shared across all service groups like blog, forum etc. then it would require configuring Marathon Load Balancer (marathon-lb) to route traffic from Azure LB to specific docker container. If we configure one ACS cluster per service group then it is an overhead to run 3-4 additional VMs required for master/slave etc.

4.	QA/Prod/Test and Blue/Green deployment

    Since ACS does not support slot concept, so in order to setup multiple environments per service for Dev, QA, Integration or Production, it would require setting up different ACS cluster per environment or scripting in a way to achieve an economic model.

    Also for production blue/green deployment model, it would require setting up another replica and script the DNS swap process after a successful deployment.

5.	JGroups Session Replication

    The problem of getting IP address of the node running Wildfly is somewhat easier to solve in Docker container model. The overlay network drivers like Flannel (https://github.com/coreos/flannel) can provide a routable IP address to each node. But it requires installing and configuring this driver on Slave nodes.

### 2.6 Azure VM Scale Set

Sometimes it feels like we can run our own Docker container management framework like Kubernates or Nomad on Azure VM Scale Set and avoid dependency on Azure. While this is definitely an option, it would require a significant level of effort in automation and monitoring and we would be treating Azure more like any other old school data center.

Considering all these limitations on Azure for Docker container support and given the fact that for running 2-4 nodes of your application, it would require running additional VMs for master nodes. Wouldn’t it be more economical to run Wildfly on native lightweight VMs using Ansible/Puppet?

### 2.7 MS SQL Server Database

Even though applications using JPA should work on any SQL database as long as we have appropriate JDBC driver but we noticed some query error on MS SQL Server database using Microsoft JDBC driver. Since not many folks are running JPA on SQL Server, these kinds of error may not be known to the community. Running a PostgreSQL server in Azure VM would require an effort to provide HA/Failover, backup and restore.

## 3. Conclusion

The Azure cloud works great for .NET applications but for Java based applications besides all the effort made by Azure, it would still require a lot of churning to get around issues one after the other. Some issues are due to lack of documentation, some are due to esoteric nature of Azure and some are due to limitations as first pass was made for .NET applications. 

Additionally, the Java applications cannot make use of some of the services provided by Azure, for example, solving session replication etc. because they fit natively in .NET model but are not widely available for various Java/JEE containers.

## References

* [http://stackoverflow.com/questions/37011980](http://stackoverflow.com/questions/37011980)
* [http://stackoverflow.com/questions/37423934](http://stackoverflow.com/questions/37423934)
* [http://stackoverflow.com/questions/37143775](http://stackoverflow.com/questions/37143775)
* [http://stackoverflow.com/questions/37663500](http://stackoverflow.com/questions/37663500)





