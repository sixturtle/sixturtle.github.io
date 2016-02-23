---
layout: post
title:  "One container for multiple environments"
date:   2015-03-20 14:05:48 -0500
categories: blog
description: Java, JEE, QA, PROD, DEV, Configuration, Docker Container, JBoss Wildfly
---
Docker container provides a better option for packaging an application as a 
micro-service. It also provides various options to customize the container
at runtime via environment variables during `docker run` command.

But what if we need to give this container to an external team so that they
can get the ready-made environment and focus on using it to build something bigger
rather than struggling with the environment setup. 

Related to above argument, what if we want to promote the same container which
was tested by Dev, to the QA environment and then promote it to Production
environment.

Well, looks like the container packaging the software bundle is the same
but the external properties, connection strings and passwords tend to
differ from one environment to the other.

What if we can package our software in a container in a way that it works
fine with defaults but does not do anything useful unless certain environment
specific properties are overridden. This is exactly what we are going to do
in this article.

Now one may point out that the Docker run command line option does provide
a way to override environment variables. Well, yes it does but there is more
to it. Often our services are built using external property files specific
to one integration. There could be multiple property files in addition to 
database connection strings etc. We do not want to checkin those environment
sensitive data into SCM repository.

In this article we are going to build a `Wildfly Docker Container` with
the `standalone.xml` customized with the environment variables rather
than actual values. Then using a mapped volume, we will pass the values
of those environment variables rather than from the docker run command line.
The advantage of this approach is that the sensitive information is not
in the SCM repository but maintained at the server level in a secured manner.


# Application Packaging


**standalone.xml**

{% highlight xml linenos %}
<?xml version='1.0' encoding='UTF-8'?>
<server xmlns="urn:jboss:domain:2.2">
    ...
    <system-properties>
        <property name="app.security.enabled" value="${env.APP_SECURITY_ENABLED}"/>
        <property name="app.config.property" value="${jboss.server.config.dir}/myapp-config.properties"/>
    </system-properties>

    <subsystem xmlns="urn:jboss:domain:datasources:2.0">
        <datasources>
            <datasource jndi-name="java:jboss/datasources/myds" pool-name="myds-pool" enabled="true" use-java-context="true">
                <connection-url>jdbc:postgresql://${env.POSTGRES_SERVER_ADDR}:${env.POSTGRES_SERVER_PORT}/mydb</connection-url>
                <driver>pgsql</driver>
                <pool>
                    <min-pool-size>5</min-pool-size>
                    <max-pool-size>20</max-pool-size>
                </pool>
                <security>
                    <user-name>${env.POSTGRES_USERNAME}</user-name>
                    <password>${env.POSTGRES_PASSWORD}</password>
                </security>
                <validation>
                    <validate-on-match>false</validate-on-match>
                    <background-validation>false</background-validation>
                </validation>
                <statement>
                    <share-prepared-statements>false</share-prepared-statements>
                </statement>
            </datasource>
            <drivers>
                <driver name="pgsql" module="org.postgresql">
                    <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
                </driver>
            </drivers>
        </datasources>
    </subsystem>
    ...
</server>

{% endhighlight %}

**Dockerfile**

{% highlight bash linenos %}

FROM jboss/wildfly:8.2.0.Final

# Setup a distribution directory for app specific files
USER root

# Anything that is not environment specific can be packaged to the container
# with a set of defaults so that if the container is handed over to someone
# it will run but may not do anything useful.
# The environment sensitive data can be overriden via /opt/dist/wildfly/bin/init.d/wildfly.conf
ADD ./deployment/wildfly $JBOSS_HOME/

# Map your Wildfly overlay here
RUN mkdir -p /opt/dist/wildfly

COPY ./wildfly-init-redhat.sh /
RUN chmod +x /wildfly-init-redhat.sh

USER jboss
ENTRYPOINT ["/wildfly-init-redhat.sh"]

{% endhighlight %}

The Docker container above is using `ADD` command to overlay build machine's local
wildfly folder to the existing Wildfly installation. This way we can package a set of
custom modules, a custom standalone.xml and application WAR/EAR files by maintaining
the same directory structure.

**Software Bundle**

{% highlight bash %}

 +/deployment
 +---/wildfly
 +------/bin/init.d
 +------------wildfly.conf    # contains environment defaults 
 +------/modules/modules/system/layers/base
 +---------/org/postgres/main
 +-------------module.xml
 +-------------postgresql-9.4-1204.jdbc42.jar
 +------/standalone
 +---------/configuration
 +------------standalone.xml # template with environment specific variables rather than actual values
 +------/deployment
 +------------your-app.war

{% endhighlight %}


In the above code, we noticed that the docker container is provided with an
entry point script. This script is nothing but a customized version of the
Wildfly provided service script which can be found at `$JBOSS_HOME/bin/init.d/wildfly-init-redhat.sh`
The init script that we use to run Wildfly as a services has the same concept 
of overriding the environment variables by sourcing `$JBOSS_HOME/bin/init.d/wildfly.conf`
file. We can leverage the same model for the Docker container as well with the
following version of `wildfly-init-redhat.sh`.

**wildfly-init-redhat.sh**

This modified version of the Wildfly init script has an additional step for 
deployment of wildfly folder using overlay mechanism. We can use this to override
Wildfly files in each environment by mapping a local path to `/opt/dist/wildfly`
path in the container. At the boot time, this docker entry point script will copy
all the files from the mapped volume to the $JBOSS_HOME directory resulting in a
custom container for the environment without rebuilding the container.

{% highlight bash linenos %}


#!/bin/bash
set -e

if [ -z "$JBOSS_HOME" ]; then
    JBOSS_HOME=/opt/wildfly
fi
export JBOSS_HOME

if [ -z "$JBOSS_USER" ]; then
    JBOSS_USER=jboss
fi

# Deploy artifacts and overlay customization from a mapped distribution folder
APP_DIST_DIR=/opt/dist/wildfly
if [ -d $APP_DIST_DIR ] && [ "$(ls -A $APP_DIST_DIR)" ]; then
    cp -r $APP_DIST_DIR/* $JBOSS_HOME
fi


# Load Java configuration.
[ -r /etc/java/java.conf ] && . /etc/java/java.conf
export JAVA_HOME

# Load JBoss AS init.d configuration.
if [ -z "$JBOSS_CONF" ]; then
    JBOSS_CONF=$JBOSS_HOME/bin/init.d/wildfly.conf
fi

[ -r "$JBOSS_CONF" ] && . "${JBOSS_CONF}"


# Set defaults.
if [ -z "$JBOSS_BIND_ADDR" ]; then
  JBOSS_BIND_ADDR=0.0.0.0
fi
 
if [ -z "$JBOSS_BIND_ADDR_MGMT" ]; then
  JBOSS_BIND_ADDR_MGMT=0.0.0.0
fi


# Startup mode of wildfly
if [ -z "$JBOSS_MODE" ]; then
    JBOSS_MODE=standalone
fi

# Startup mode script
if [ "$JBOSS_MODE" = "standalone" ]; then
    JBOSS_SCRIPT=$JBOSS_HOME/bin/standalone.sh
    if [ -z "$JBOSS_CONFIG" ]; then
        JBOSS_CONFIG=standalone.xml
    fi
else
    JBOSS_SCRIPT=$JBOSS_HOME/bin/domain.sh
    if [ -z "$JBOSS_DOMAIN_CONFIG" ]; then
        JBOSS_DOMAIN_CONFIG=domain.xml
    fi
    if [ -z "$JBOSS_HOST_CONFIG" ]; then
        JBOSS_HOST_CONFIG=host.xml
    fi
fi


if [ "$JBOSS_MODE" = "standalone" ]; then
    $JBOSS_SCRIPT -Djboss.bind.address=$JBOSS_BIND_ADDR -Djboss.bind.address.management=$JBOSS_BIND_ADDR_MGMT -c $JBOSS_CONFIG 
else 
    $JBOSS_SCRIPT -Djboss.bind.address=$JBOSS_BIND_ADDR -Djboss.bind.address.management=$JBOSS_BIND_ADDR_MGMT --domain-config=$JBOSS_DOMAIN_CONFIG --host-config=$JBOSS_HOST_CONFIG 
fi

{% endhighlight %}

# Running container in different environments

Now imagine running above container in a specific environment where database 
connection string is different, properties are different. We could just setup
a directory structure like below in the local machine and map it to the container
at Docker run time.

**Build container**

{% highlight bash %}

# Example:
$ docker build --rm -t <username>/wildfly-ex .

{% endhighlight %}

**Setup environment specific volume**

{% highlight bash %}

+/opt
+--/deployment
+------/dev
+----------/wildfly
+------------/bin/init.d
+----------------wildfly.conf    # contains environment specific sensitive values
+------------/standalone
+--------------/configuration
+----------------myapp-config.propertie  # contains environment specific sensitive values
+------/qa
+----------/wildfly
+------------/bin/init.d
+----------------wildfly.conf    # contains environment specific sensitive values
+------/prod
+----------/wildfly
+------------/bin/init.d
+----------------wildfly.conf    # contains environment specific sensitive values

{% endhighlight %}

**Run container with the mapped volume**

{% highlight bash %}

# Environment: DEV
$ docker run -d -p 8080:8080 -v /opt/deployment/dev/wildfly:/opt/dist/wildfly --name wildfly-dev <username>/wildfly-ex
$ docker stop wildfly-dev
$ docker start wildfly-dev

# Environment: QA
$ docker run -d -p 8080:8080 -v /opt/deployment/qa/wildfly:/opt/dist/wildfly --name wildfly-qa <username>/wildfly-ex
$ docker stop wildfly-qa
$ docker start wildfly-qa

{% endhighlight %}

In summary, the build engines like Jenkins or Bamboo should be used to package 
the software in the container with default values that is independent of any
environment and then at run time, the environment specific data should be
provided to the container via a mapped volume.
