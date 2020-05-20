+++
title = "Using CDI 2.0 in a Java SE application"
date = "2019-08-29"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Java"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

CDI (Context & Dependency Injection) is a Java API released with JEE6 that enable dependency injection. Prior to may 2017 it was only available on JEE platform, but fortunately it has changed.

CDI 2.0 (released in may 2017) add a new API to create a dependency injection container on a Java SE application. And the integration is easy. Here is how to do it:

NB: This project was written in Java 11 with maven as dependency management.

# Setup maven dependencies

The easy way to get started is to use the following dependency:

```xml
<dependency>
    <groupId>org.jboss.weld.se</groupId>
    <artifactId>weld-se-shaded</artifactId>
    <version>3.1.2.Final</version>
</dependency>
```

weld-se-shaded is an "uber jar" that package all needed dependencies to enable CDI on Java SE (cdi-api, weld-core, ...).

# Create the beans

We need to declare the CDI beans:

First we need create a MyService interface to abstract bean interactions:

```java
package service;

public interface MyService {
   void sayHello(String username);
}
```

Then we implement the interface to define the service behavior:

```java
package service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.enterprise.context.Dependent;

@Dependent
public class MyServiceImpl implements MyService {

    @PostConstruct
    public void initialize() {
        System.out.println("Initializing");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Cleaning");
    }

    public void sayHello(String username) {
        System.out.println("Hello " + username + " from " + MyServiceImpl.class.getName());
    }
}

```

The bean is marked as **@Dependent**. This mean that the bean lifecycle is bound to the lifecycle of the bean that inject it. The instance is not shared with any other beans.

The **@PostConstruct** and **@PreDestroy** annotations are lifecycle annotations used as "Constructor" / "Destructor", they are called by the container when the bean is instantiated and processed, and when the bean is released from the container.

Next we create a StartupService that will inject MyService and use it:

```java
package service;

import javax.annotation.PostConstruct;
import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;

@ApplicationScoped
public class StartupService {

    @Inject
    private MyService myService;

    public void sayHello() {
        myService.sayHello("world");
    }
}

```

The bean is declared **@ApplicationScoped**. This mean that the bean will act as a singleton and will live as long as the CDI container is running. In this bean I inject MyService and use it in the sayHello method.

# Bootstrap the container

Then we'll need to bring up the CDI container. It is generally done on the main method of your program.

```java
import org.jboss.weld.environment.se.Weld;

public class MainApplication {

    public static void main(String[] args) {
        Weld weld = new Weld();
    }
}

```

**We are not done yet**! Now you need to initialize the container. But before doing so you'll need to choose how to register the beans. This can be done mainly in two ways:

- Using runtime scanning: the container will search for bean in your archive and register them.
- Programatically: you can register manually the bean using Weld#addBeanClass or Weld#addBeanClasses


In this example I will chose runtime scanning. Please note that this may introduce performance issues at startup when dealing with big applications. But this is not a big issue since you can restrict the scan on specifics packages, or use jboss-yandex instead of the built-in Java Reflection to optimise performances.

Now, we initialise the container (using a try with-resources to prevent any resources leak) and we call the sayHello method of the StartupService.

```java
import org.jboss.weld.environment.se.Weld;
import org.jboss.weld.environment.se.WeldContainer;

import service.StartupService;

public class MainApplication {

    public static void main(String[] args) {
        Weld weld = new Weld();
        try (WeldContainer weldContainer = weld.initialize()) {
            weldContainer.select(StartupService.class).get().sayHello();
        }
    }
}

```

# Declare the beans

Since we are using runtime scanning and we do not programatically specify a scanning strategy, we need to create a beans.xml file in the META-INF directory. This file will be used by the CDI container to declare the discovery mode.

```xml
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/beans_2_0.xsd"
       version="2.0" bean-discovery-mode="annotated">
</beans>
```

the bean discovery mode can have the following values:

- all: all java bean/classes will be registered as CDI bean if possible and may be injected. Their scope will be dependent by default.
- annotated: the container will only register bean/classes with CDI annotations as CDI bean. Their scope should be explicitly defined otherwise they won't be registered.

# Running the app

```text
Aug 29, 2019 1:10:20 PM org.jboss.weld.bootstrap.WeldStartup <clinit>
INFO: WELD-000900: 3.1.2 (Final)
Aug 29, 2019 1:10:20 PM org.jboss.weld.environment.deployment.discovery.ReflectionDiscoveryStrategy processAnnotatedDiscovery
INFO: WELD-ENV-000014: Falling back to Java Reflection for bean-discovery-mode="annotated" discovery. Add org.jboss:jandex to the classpath to speed-up startup.
Aug 29, 2019 1:10:21 PM org.jboss.weld.bootstrap.WeldStartup startContainer
INFO: WELD-000101: Transactional services not available. Injection of @Inject UserTransaction not available. Transactional observers will be invoked synchronously.
Aug 29, 2019 1:10:22 PM org.jboss.weld.environment.se.WeldContainer fireContainerInitializedEvent
INFO: WELD-ENV-002003: Weld SE container 375aad40-5b62-405c-b948-f612027d2cce initialized
Initializing
Hello world from service.MyServiceImpl
Cleaning
Aug 29, 2019 1:10:22 PM org.jboss.weld.environment.se.WeldContainer shutdown
INFO: WELD-ENV-002001: Weld SE container 375aad40-5b62-405c-b948-f612027d2cce shut down
```

you can see that the CDI container is successfully bootstraped and our StartupService is called and triggered  the method sayHello on our MyService. The lifecycle methods are working too since we see the Initialising and Cleaning output.

Happy hacking!