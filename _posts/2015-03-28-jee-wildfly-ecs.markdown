---
layout: post
title:  "Deploy JEE Wildfly Container in ECS manually"
date:   2015-03-28 10:05:48 -0500
categories: blog
description: Java, JEE, EC2 Container Service, Configuration, Docker Container, JBoss Wildfly, Amazon Elastic Beanstalk
---
Amazon EC2 Container Service ([ECS]) is a highly scalable, high performance 
container management service that supports Docker containers and allows you 
to easily run applications on a managed cluster of Amazon EC2 instances. 

In this article we are going to deploy the container described in the other [article]
to AWS ECS. The purpose of this article is to understand ECS components so that
we better understand Elastic Beanstalk.

* Create ECS Cluster (mycluster)
* Setup S3 Bucket (mys3bucket) with ecs.config

    {% highlight bash %}
    $ vi ~/ecs.config
        # set environment variables for Cluster name and Docker Repository credentials
        ECS_CLUSTER=mycluster
        ECS_ENGINE_AUTH_TYPE=dockercfg
        ECS_ENGINE_AUTH_DATA={"https://your-docker-repo/":{"auth":"xxxxxxxxxxxxxxxxx","email":"you@domain.com"}}

    $ aws s3 cp ecs.cfg s3://mys3bucket/
    {% endhighlight %}

* Setup IAM ecsInstanceRole with policies to provide access to S3 and ECS (AmazonS3ReadOnlyAccess & AmazonEC2ContainerServiceforEC2Role)
* Setup IAM ecsServiceRole with policies to provide access to ELB (AmazonEC2ContainerServiceRole)
* Setup a security group (instance-sg-ec2) with SSH and port 8080 access
* Setup another security group (elb-sg) with port 80 and mapping to EC2 instance at port 8080
* Launch an EC2 instance using an ECS Optimized Linux Image (amzn-ami-2015.09.d-amazon-ecs-optimized/ami-ac6872cd) - image should have correct version of Docker and ECS agent

   - Choose ecsInstanceRole
   - Choose instance-sg-ec2
   - Using User Data, provide initial script to install AWS CLI and download ecs.config from S3
    {% highlight bash %}
        #!/bin/bash
        sudo yum install -y aws-cli
        sudo aws s3 cp s3://mys3bucket/ecs.config /etc/ecs/ecs.config --region us-west-2
    {% endhighlight %}

* Once EC2 instance is up, it should join with the ECS Cluster based on the cluster name
* (Optional) Create an ELB (myelb) with security group (elb-sg) and health check mapped to service to be running in EC2

* Create ECS Task Definition - (myTask)
* Create ECS Service (mySvc) with a number of Task instances (myTask) and link with optional ELB (myelb) - choose the container exposing port 8080
* Wait for ECS service to schedule and run task on EC2 Container Instances (Pending to Running) - monitor events coming from ecs-agent
* Access your service via ELB
* Access/Troubleshoot EC2 Container instance

    - ssh to instance and you will notice docker, aws cli and ecs agent are installed
    - docker ps
    - sudo start or stop ecs
    - cat /etc/ecs/ecs.config
    - cat /var/log/ecs/ecs-agent-xxxxxx
    - ECS Agent introspection: curl http://localhost:51678/v1/metadata


[ECS]: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html "Amazon EC2 Container Service"
[Dockerfile]: https://github.com/sixturtle/examples/tree/master/docker/wildfly-ex "Wildfly Dockerfile"
[Container]: https://hub.docker.com/r/sixturtle/wildfly-ex/ "Docker Container"
[article]: /blog/2015/03/20/one-container-many-envs.html
