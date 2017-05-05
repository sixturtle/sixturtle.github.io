---
layout: post
title:  "How to profile Wildfly using NetBeans and JMeter"
date:   2017-05-05 10:05:48 -0500
categories: blog
description: Java, JEE, JBoss Wildfly, NetBeans, Memory Leak, Performance Issue, JMeter, Profiling 
comments: true
---
The NetBeans IDE includes a powerful profiling tool that can provide important information about the runtime behavior of your application. The NetBeans profiling tool easily enables you to monitor thread states, CPU performance, and memory usage of your application from within the IDE, and imposes relatively low overhead.

## 1. Download NetBeans IDE
The first step in the setup is of course the NeaBeans IDE.
[Download](https://netbeans.org/downloads/index.html) and install the basic version of IDE which already includes Profiler.

## 2. Configure Java Agent
In general, a javaagent is a JVM "plugin", a specially crafted .jar file, that utilizes the [Instrumentation API](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html) that the JVM provides.

When you click on Profile menu of the NetBeans IDE, it will prompt you to setup attach settings, which looks like following.

* Step 1: Make sure the target application is configured to run using Java 6+. Click to update steps for profiling JDK 5 applications.

* Step 2: If you have not done it before create a Remote profiling pack for the selected OS & JVM and upload it to the remote system. Remote profiling pack root directory will be referred to as <remote>.

* Step 3: If you have not run profiling on the remote system yet, run the <remote>/bin/calibrate-16.sh script first to calibrate the profiler.

* Step 4: Add the following parameter(s) to the application startup script (copy to clipboard):

         -XX:+UseLinuxPosixThreadCPUClocks -agentpath:<remote>/lib/deployed/jdk16/linux-amd64/libprofilerinterface.so=<remote>/lib,5140

* Step 5: Start the target application. The process will wait for the profiler to connect.

* Step 6: Submit this dialog and click the Attach button to connect to the target application and resume its execution.


The IDE generates different Agent for different OS & JVM. Please make sure you select the right OS, e.g. 32 bit vs 64 bit. Download the generate agent zip file and copy it to the remote server that you want to profile.


### 2.1 Add agent to Wildfly startup
When you start Wildfly with the NetBeans Profiling Agent, the JBoss Logger throws an error with the message to configure logging manager like this - `-Djava.util.logging.manager=org.jboss.logmanager.LogManager`. Following steps correctly configure the JAVA_OPTS for both - logging and agent.

1. Unzip agent to a directory, say `/opt/profiler/`.
2. Edit `$JBOSS_HOME/bin/standalone.conf` file to augment `JAVA_OPTS`

        # make sure you comment out existing byteman coming from $JBOSS_MODULES_SYSTEM_PKGS
        #JAVA_OPTS="$JAVA_OPTS -Djboss.modules.system.pkgs=$JBOSS_MODULES_SYSTEM_PKGS -Djava.awt.headless=true"
        JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"

        # Configure JBoss Log Manager (version may be different on your system)
        JAVA_OPTS="$JAVA_OPTS -Xbootclasspath/p:$JBOSS_HOME/modules/system/layers/base/org/jboss/logmanager/main/jboss-logmanager-2.0.3.Final.jar:$JBOSS_HOME/modules/system/layers/base/org/jboss/log4j/logmanager/main/log4j-jboss-logmanager-1.1.2.Final.jar"
        JAVA_OPTS="$JAVA_OPTS -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
        JAVA_OPTS="$JAVA_OPTS -Djboss.modules.system.pkgs=org.jboss.byteman,org.jboss.logmanager"

        # Configure NetBeans Agent
        JAVA_OPTS="$JAVA_OPTS -XX:+UseLinuxPosixThreadCPUClocks -agentpath:/opt/profiler/lib/deployed/jdk16/linux-amd64/libprofilerinterface.so=/opt/profiler/lib,5140"

3. Start the server

        $JBOSS_HOME/bin/standalone.sh


## 3. Attach Profiler
Once you start Wildfly Server with the agent, it will be waiting for the NetBeans IDE to connect. From the
NetBeans IDE attach to the remote server by specifying the IP address or hostname of the server.

That's all needed to start the monitoring.

The NetBeans IDE allows multiple types of profiling. Please make sure you select `Enable Multiple Modes` to see more than one at a time.

* Telemtry - includes graphs for GC & CPU, Heap Usage, Threads etc.
* Objects - include live byte and live count of object - very helpful for identifying memory leak
* Methods - includes time take by each method call - very helpful for identify performance bottleneck
* Threads
* Locks
* SQL Queries


## 4. Run load test using JMeter
You are profiling the application because under certain load, the application is showing either performance issue or memory leak. You would need to prepare a JMeter test suite to generate similar load on your server connected to the NetBeans IDE.

[Apache JMeter](http://jmeter.apache.org/) can be used to simulate a heavy load on a server, group of servers, network or object to test its strength or to analyze overall performance under different load types.

Follow the JMeter guide to learn how to script your test cases. Once Test suite is ready, run it for the kind of load you want to generate on your server.

## 5. Profling Snapshots
When JMeter load test is running, the NetBeans profiler shows the live statistics. From time to time, you can take snapshot of the stats so that you can compare two snapshots to find difference in stats.

The Snapshots or live metrics allow you to filter for a package or class. Since Wildfly server includes so many jars, it is important to right click and filter the package that you want to debug. There are few obvious areas to inspect for performance and memory leak -

 * javax.ws - For Web Service request/response
 * java.net - For Sockets
 * org.infinispan - For cache size monitoring

The infinispan cache [configuration](https://docs.jboss.org/author/display/WFLY10/Infinispan+Subsystem) in `$JBOSS_HOME/standalone/configuration/standalone.xml` may require tunning to limit the max cache entries and eviction, expiration policy. These settings are critical to limit the growing Heap size.

Sometimes it is not obvious that you would need to explicitly close a resource before losing the object reference, otherwise it will lead to a memory leak. For example -

    HttpClient client = session.getProvider(HttpClientProvider.class).getHttpClient();
    HttpResponse response = client.execute(post);
    //Consume the entity if you need to
    HttpClientUtils.closeQuietly(response);

This ensures that regardless of if you consume the entity or not it will be properly closed.

## 6. Miscellaneous
Since all 3 components - NetBeans IDE, JMeter Load Tests and Wildfly Server require significant amount of memory, you would need a high compute and dedicated computer to run all these. The whole process of identifying the issue using load test and profiling may take several hours to days of effort.

