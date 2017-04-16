---
layout: post
title:  "Deploy JEE Wildfly Container in ECS using Elastic Beanstalk"
date:   2015-03-28 14:05:48 -0500
categories: blog
description: Java, JEE, QA, PROD, DEV, Configuration, Docker Container, JBoss Wildfly, Amazon Elastic Beanstalk
comments: true
---
Amazon [Elastic Beanstalk] provides various pre-configured application 
environments for rapid application development. Among those are Python,
Ruby, PHP, Node.js Java, .NET and Docker. The Java environment is very 
specific to Tomcat deployment and there is no environment available 
for JEE containers like Wildfly, WebLogic or Glassfish. In this article 
we are going to use Docker container for Wildfly to deploy a JEE application
and manage environment specific sensitive information outside of SCM.


# 1. Docker container for Wildfly application   

There is a great [article] on how to build an environment independent Docker 
container for JEE application bundle to run on Wildfly. The [Dockerfile] is 
available on GitHub and the [Container] is available on Docker Hub. We are going 
to use this container for deployment using Elastic Beanstalk.


# 2. Dockerrun.aws.json file to launch container   

Once we have a Docker container, we can run it locally using `Docker run`
command. This is exactly what we do for local development and debugging.

For deploying a container to AWS, someone needs to execute `Docker run` for
our container. Well, Amazon provides EC2 container services (ECS) to manage
Docker containers in an EC2 instance. For that there are special AMIs packaged
with an ECS Agent (another container). The ECS agent ensures that the container
keep running as per the HA requirements in the Task Definition.
It also provides diagnostics and status information to the Elastic Load Balancer
etc.

Elastic Beanstalk uses Dockerrun.aws.json file to provide launch configuration 
for a container.

{% highlight json %}

{
    "AWSEBDockerrunVersion": 2,
    "authentication": {
        "bucket": "<your-s3-bucket-name>",
        "key": "<your-docker-authentication-json-file-name>"
    },
    "volumes": [
        {
          "name": "distribution",
          "host": {
            "sourcePath": "/opt/deployment/wildfly"
          }
        }
    ],
    "containerDefinitions": [
     {
        "name": "wildfly-ex",
        "image": "sixturtle/wildfly-ex",
        "essential": true,
        "memory": 512,
        "portMappings": [
          {
            "hostPort": 8080,
            "containerPort": 8080
          }
        ],
        "mountPoints": [
            {
              "sourceVolume": "distribution",
              "containerPath": "/opt/dist/wildfly",
              "readOnly": false
            }
        ]
     }
    ]
}

{% endhighlight %}

Here is the breakdown structure of the above Dockerrun file.

**Docker Repository Authentication**

If we have our container hosted in a private repository then we need to provide
authentication information to ECS agent to pull the image. The docker encrypted
version of credential can be obtained from `~/.docker/config.json`. This file
gets created automatically once we execute command `$ docker login` successfully.

For Dockerrun authentication we need to trim the default docker config.json file
to make it look like this - 

{% highlight json %}
{
    "https://index.docker.io/v1/": {
        "auth": "TY6UjwouUSDMUtlbXAzMjAy",
        "email": "<your-email-address>"
    }
}
{% endhighlight %}

Next, we need to upload this file to an S3 bucket and provide bucket information
in the Dockerrun file as above. The EC2 instance must have proper IAM role attached
so that it can access the S3 bucket.

**Container Launch Configuration**

The `containerDefinitions` section of the Dockerrun file includes Docker container
image name as it appears in the Docker repository and also includes `portMappings`
section similar to what we would provide to `Docker run -d -p port:port` command
locally.

As we know that the Docker run provides various other options to launch container
in a specific way, most of those options can be provided in the Dockerrun file.
We are going to use volume mount option to keep our environment specific data
outside of the container but in the local EC2 instance.

The `volumes` section of Dockerrun file defines a path in EC2 instance with a tag
name to be used in the `mountPoints` sub-section of the `containerDefinitions` 
section. The`mountPoints` section maps a local EC2 instance path to a path in the
Docker container.

As specified in this [article], we are mapping a volume that contains environment
sensitive data.

**Environment Sensitive Data in S3**

Now, the question is how would an EC2 instance get my environment specific data?
Well, to keep the environment sensitive data in a protected area, we can use an S3
bucket and configure Elastic Bean using `.ebextensions` file to download the files
from S3 bucket at the time of launching EC2 instances.

We need to create a file with extension .config in project's .ebextensions folder
with following content.

{% highlight yaml %}
files:
    "/opt/deployment/wildfly/bin/init.d/wildfly.conf":
        mode: "000644"
        owner: root
        group: root
        source: https://s3-us-west-2.amazonaws.com/your-bucket-name/wildfly/bin/init.d/wildfly.conf

Resources:
  AWSEBAutoScalingGroup:
    Metadata:
      AWS::CloudFormation::Authentication:
        S3Auth:
          type: "s3"
          buckets: ["your-bucket-name"]
          roleName:
            "Fn::GetOptionSetting":
              Namespace: "aws:asg:launchconfiguration"
              OptionName: "IamInstanceProfile"
              DefaultValue: "aws-elasticbeanstalk-ec2-role"
{% endhighlight %} 

The `files` section of .ebextensions file is used to copy a file into EC2 instance from a 
location as specified under `source`. The `Resources` section provides a way to authenticate
with S3. Again the EC2 instances should have the same IAM role attached as specified in the 
resources section.

# 3. Elastic Beanstalk Deployment

Once we have two files - Dockerrun.aws.json and .ebextensions/env.config, we can zip them to
create what EB calls an application source bundle. 

`$ zip ../myapp.zip -r * .[^.]*`

Login to AWS EB console and follow the application creation wizard to deploy this application.

We can freely checkin above two files in SCM as they do not contain any sensitive information.


# 4. Troubleshooting

In order to troubleshoot above configuration, we need to SSH to the EC2 instance  
assigned to the ECS Cluster. Once we are logged-in to the EC2 instance, we 
can explore following.

{% highlight bash %}
# The application source bundle location
$ cd /var/app/current/

# The S3 downloaded files
$ cd /opt/deployment/wildfly

# Docker container
$ sudo docker ps -a
$ sudo docker logs <container-id>
$ sudo docker exec -it <container-id> bash
{% endhighlight %}

We should be able see the mapped volume content inside container path /opt/dist/wildfly.
The docker entry point script should have copied files from /opt/dist/wildfly to 
$JBOSS_HOME path within container before starting Wildfly process.


[Elastic Beanstalk]: https://aws.amazon.com/elasticbeanstalk/getting-started/ "Amazon Elastic Beanstalk"
[Dockerfile]: https://github.com/sixturtle/examples/tree/master/docker/wildfly-ex "Wildfly Dockerfile"
[Container]: https://hub.docker.com/r/sixturtle/wildfly-ex/ "Docker Container"
[article]: /blog/2015/03/20/one-container-many-envs.html
