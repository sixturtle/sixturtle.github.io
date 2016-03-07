---
layout: post
title:  "JEE Development without Spring Framework"
date:   2015-04-06 10:05:48 -0500
categories: blog
description: Java, JEE, Spring Framework, REST, JAX-RS, JPA, ORM, Undertow Servlet, RestEasy, DbUnit, WireMock
---
This is an interesting article as it compares Spring Framework with
Java Enterprise Edition. It may become a debatable topic and may upset many
Spring Framework users. Well, to be fair, I want to let you know that I am
one of the biggest fan of Spring Framework but yet I am writing this article
because I believe we don't always need Spring Framework for Java development.
If you want to see a proof, checkout this [**project**](https://github.com/sixturtle/examples/tree/master/jee-fuse).

Q: When Java EE has everything then why do we choose Spring MVC? Is it - 

* For IoC ?
* For RESTful ?
* For jUnit
* For JPA ?
* To allow running in a Servlet Container, e.g. Tomcat ?

Many times, I have noticed that it is a cultural preference over any valid 
technical reason of choosing Spring Framework. There was a time when JEE was
not up the mark and developing using Spring Framework was much more productive.
But is this the case today? Today JEE includes but not limited to following -

* Context and Dependency Injection (JSR 299) 
* JAX-RS (JSR 339)
* Java Persistence API (JSR 317)
* Java Transaction API (JSR 907) 
* Bean Validation (JSR 303) 
* Java Authentication and Authorization Framework (JAAS) – JSR 196
* Java Logging API (JSR 47)
* and many more

The JEE certified containers are expected to provide a concrete implementation
of above specifications. Majority of these implementations are still sourced
from open source community that the users of Spring Framework are accustomed to.

So the question is why to put effort in picking up various implementations to 
create a custom bundle when we already have a set of certified containers like - 

* RedHat JBoss AS
* Wildfly (Open Source)
* Glassfish (Reference Implementation)
* Oracle WebLogic Server
* IBM  Websphere Application Server
* and many more

Ah, I hear this one a lot, JEE containers are heavy weight and for our purpose
Tomcat or Jetty is just enough. Well, may be it is a knowledge gap. The Wildfly
container is as light as Tomcat. We can add or remove subsystems as we need to
add or remove features. Both Wildfly and WebLogic startup time is in milliseconds.
The classloader of Wildfly container does not load all the services but a minimum
set needed by the application.

Those who are experienced and manage the runtime libraries accurately without a
conflict, it does not matter whether they use pure JEE or Spring Framework because
most likely they will code against JEE API even if they use Spring Framework with
the exception of one or two highly proclaimed features of Spring. Those proclaimed
implementations can be brought to a JEE container in the same way as it is used
in Spring + Tomcat.

But when naive programmers try to use Spring Framework, they usually start 
adding dependencies to the project without understanding the side effects. The end
result is a set of conflicting dependencies, multiple versions of the same JAR.
It works in one programmer's Eclipse but does not work in others. And finally
it behaves differently in Production.

We cannot live with uncertainties. We must know what set of dependencies we
need and why. We cannot introduce 3 more indirect dependencies when we needed
one utility function from an open source library.

In Wildfly (or JBoss AS), the dependencies are very well managed by a set of modules
and their association with other modules. The JEE container provides implementation
to all the JEE specifications and passed the certification test to comply with
the spec. We can code just against those standard JEE APIs without worrying about
runtime implementations, let those be managed by a certified container. This would
let us focus on making use of Java language rather than dealing with a framework
issues. The code would be potable, consistent and well understood by any other
Java engineer faster than the code developed with many custom frameworks.

To see an example of pure Java implementation of a REST API with full jUnit and
mocking support, please see [code here](https://github.com/sixturtle/examples/tree/master/jee-fuse).
This code also produces a nice Swagger API documentation without polluting Java
code with Swagger annotations.

## Advantages of using Spring Framework

* Spring Framework works great in many ways - great community support, faster upgrades available
* Great functional test support for web/db/ws contexts – otherwise harder to test
* Great support for properties externalization
* Self contained and can run in either a JEE certified container or in a servlet container

## Disadvantages of using Spring Framework

* Spring Framework is not JEE – it provides integration with services offered for JEE or custom framework
* Vendor lock
* Mix of technologies but each one has its own release cycle
* JEE is changing fast to standardize the best of bests available in the community – every major release of Spring Framework is a big change to catch up with Java
* Upgrade to newer version is not a simple JAR upgrade but lots of code change if we need to take advantage of newer features

## Advantages of using a JEE Standard APIs

* Portable code
* No vendor lock
* Move along Java EE upgrades
* Using another part of JEE (e.g. Messaging, Async processing, Security) would be any easy addition and would work smoothly
* Standard monitoring provided by containers like Jboss
* Take advantage of standard based services and annotations provided by certified JEE Containers

## Disadvantages (myth) of using a JEE standard APIs

* jUnit for container based services are hard, e.g. IoC/DI
* JEE containers are heavy weight

In summary, when I am not using Spring Framework, I don't lose all the cool 
parts of Spring. For example, if I liked the generic JPA adapter in Spring
Framework, I can just consider that as a design pattern or utility and copy
the source code to my project. This way I am not reinventing the wheel but
still in control of standards and dependencies.
