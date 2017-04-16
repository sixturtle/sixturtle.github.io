---
layout: post
title:  "Setup Amazon VPC with Public and Private Subnets"
date:   2015-03-15 14:05:48 -0500
categories: blog
description: Java, JEE, Amazon Virtual Private Cloud, VPC, Networking, Subnets, Public, Private
comments: true
---
Setting up a virtual private cloud (VPC) in Amazon Web Services (AWS) is not
supposed to be difficult if we understand following instructions from Amazon
However, there are couple of minor gotchas that may result in a non-functional
VPC. In this article, we will setup a fully functional VPC and deploy a two
tiered application.

* [Amazon Elastic Beanstalk Guide - VPC Basic](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/vpc-basic.html)
* [Amazon VPC Guide](http://docs.aws.amazon.com/AmazonVPC/latest/GettingStartedGuide/ExerciseOverview.html)


# 1. VPC Model
    
Suppose we have a multi-tiered application which needs to be deployed as follows.

    VPC Network (10.0.0.0/16)
        public subnet (10.0.0.0/24)  - AZ 1
            - Elastic Load Balancer (DNS Name)
            - NAT Instance + Bastion host (public Elastic IP)
        private subnet (10.0.1.0/24) - AZ 1
            - Web/Application Server (private IP only)
            - Amazon RDS HA Database (DNS Name)
        private subnet (10.0.2.0/24) - AZ 2
            - Amazon RDS HA Database (DNS Name)


![VPC Diagram](/res/aws-vpc.png)


From the VPC Dashboard, click `Start VPC Wizard` and choose `VPC with Public and
Private Subnets`. 

The wizard pre-populates VPC form with a default network of 10.0.0.0/16, 
a public subnet (10.0.0.0/24) and a private subnet (10.0.1.0/24). 

These CIDR blocks can be changed to use other IP address patterns, 
e.g. 172.0.0.0 or 192.168.0.0. There are various subnet calculators available
online to help configure subnets. The VPC console does subnet validation and 
would not allow invalid subnet definition.

**Subnets and Availability Zones** - Let's choose the same availability zones
for both public and private subnets.

**NAT gateway** - very critical and required piece to allow Elastic 
Beanstalk or ECS to get statistics from EC2 instances running in the private 
subnet.

`Use a NAT instance instead` option will create an EC2 instance with a NAT 
interface in the public subnet. This EC2 instance can be used as a 
bastion host to SSH to the public subnet and then jump to private subnet.

*Note*: This version of NAT gateway would not be listed under VPC Dashboard->
NAT Gateway but can be found under EC2 Dashboard->Network Interfaces.

`Use a NAT gateway instead` option needs us to provide an Elastic IP Allocation ID, 
which can be obtained from EC2 Dashboard->Elastic IP. We may need to allocate 
a new EIP for the VPC if there isn't a spare one.

**S3 Endpoint** - If EC2 instances need access to files from S3 buckets then we
need to setup an S3 endpoint. We have option to limit S3 bucket access to private
,public or both subnets.


The VPC wizard will automatically create following resources -    

* an internet gateway (IGW),   
* a NAT gateway,   
* two subnets (one public and one private),   
* two route tables (public route that connects to IGW and private route that connects to NAT gateway)   
* a default security group for VPC.   

One route table is by default associated to the public subnet and we can call it
public route. We need to attach private subnet to the other route table and call
it private route. The private route is the `main` route. The public route is
pre-configured with route to the internet gateway and the private route is 
pre-configured with route to the NAT gateway.

**Security Groups** - There should be just one default security group created for 
the VPC. The inbound rule for this VPC allows all traffic from within network. 
Meaning if we attach this security group to two EC2 instances, those two instances
can communicate to each other on any port. Let's call it a public security group
and edit the inbound rule to allow SSH connection from anywhere (0.0.0.0/0). 
*Note:* This security group is already applied to the NAT instance which is also
our bastion host.

    inbound: 
      SSH from outside network
      All traffic from within this security group (default)
    outbound:
      All traffic to anywhere (default)

Let's create one more security group and call it a private security group or 
instance security group. Make sure to pick the right VPC. We will use this
security group for EC2 instances created in the private subnet. So let's
configure the inbound and outbound rules.

    inbound: 
      SSH from public security group
      All traffic from within this security group
    outbound:
      All traffic to anywhere (default)

This completes our typical VPC setup. However, for Amazon RDS we need to augment
our VPC.


**Amazon RDS Subnet Group** -

Amazon RDS requires a minimum of two availability zones. Since we created both
subnets in the same availability zones, now we need to create another private
subnet (10.0.2.0/24) under newly created VPC but choose different availability
zone. Attach this subnet to the private route table. So private route now has 
two private subnets associated.

From the Amazon RDS Dashboard->Subnet Group, create a subnet group that would 
include two private subnets from two different availability zones. Make sure to select
the right VPC and add both private subnets. We can click on `add all the subnets`
and then remove the public subnet (10.0.0.0/24) or add one private subnet at a time.

*Note:* This RDS subnet group will be used while creating an RDS instance.


# 2. Configure Database in Private Subnet

From Amazon RDS Dashboard->Instances->Launch DB Instance->Select DB Engine, create
a database. In the `Network & Security` part of the DB creation wizard, select right VPC
and the RDS subnet group created for this VPC. Also select `private` VPC security
group created above. We have option to pick availability zone based on the subnet
group. For development, we can select just one availability zone (same as our 
public subnet's zone) to keep everything in one zone.

The wizard will create an RDS instance in the private subnet and provide the DB URL
and port number for JDBC configuration in the application/web server.


# 3. Deploy application in Private Subnet with ELB in Public Subnet

There are multiple ways to deploy an application to AWS (e.g. direct EC2 instance
, EC2 Container Services, Elastic Beanstalk etc.), but we are going to just focus
on Elastic Beanstalk deployment method using Web console. The idea is to show how
to choose VPC, private subnet and security group.

From Elastic Beanstalk Web Console follow `Create New Application->New Environment->Web
Server Environment->Create Web Sever` with following remarks.

On the `Environment Type` screen, choose predefined configuration for the application and then
select `Load Balancing & Auto Scaling` for the environment type.

On the `Additonal Resources` screen, check `Create this environment inside a VPC`
but do not check the RDS option as we have already created database.

On the `VPC Configuration` screen, select the right VPC, uncheck `Associate 
Public IP Address`, select ELB in the public subnet (10.0.0.0/24) and EC2 in the
private subnet (10.0.1.0/24), select VPC security group as private security group 
created above and select ELB external (internet) facing.

The wizard will deploy application in the ECS environment with a Load Balancer 
and Auto Scaling group.   

***Note:** The security groups are not directly tied to any subnet but are applied to 
resources like EC2 instances, ELB, Auto Scaling Group etc. The inbound and outbound
rules of the applied security group control the traffic to/from these resources.
Depending upon how these resources are created, Amazon may create new security group
and apply to the resource, especially for ELB and ASG. We need to tweak the inbound
and outbound rules of these new security groups to allow traffic flow from/to the
EC2 instances. For example, if our application server runs at port 8080 but ELB
listener is configured to map input port 80 to the instance port 8080 then we 
need to update the security group of the EC2 instance to allow traffic from ELB at
port 8080.*

***Note:** Since our private security group allows all traffic from within the same group,
the application/web server should not have any trouble connecting to the database. If
application server is unable to connect to the database then check the security groups
applied to the database instance and EC2 instance running the application. Both should
have the same security group but for some reason if we accidently let the wizard auto create
a security group for RDS then just tweak the firewall rules of RDS security group to
allow traffic from the security group of EC2 instance.*

