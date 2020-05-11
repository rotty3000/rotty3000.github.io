---
title: Gogo Shell
excerpt: The telnet client is very useful (really invaluable) for understand what's happening in the osgi runtime. OSGi is very dynamic, and due to this you really need to get live feedback about the state of things.
---

## Setting up the Telnet Client in Liferay

The telnet client is very useful (really invaluable) for understand what's happening in the osgi runtime. OSGi is very dynamic, and due to this you really need to get live feedback about the state of things.

* which modules are installed
* which packages they provide
* which services are available
* why some component isn't resolving
* etc.

## Not currently Bundled, so Install it

While the current portal does not bundle Gogo or Telnet support, it's very easy to fetch and install the parts you need:

* create a directory in your system where you can store the necessary libraries (you probably have one already)
* download these 4 libs into that directory:
  * http://archive.apache.org/dist/felix/org.apache.felix.gogo.command-0.12.0.jar
  * http://archive.apache.org/dist/felix/org.apache.felix.gogo.runtime-0.10.0.jar
  * http://archive.apache.org/dist/felix/org.apache.felix.gogo.shell-0.10.0.jar
  * http://eclipse.mirror.rafal.ca/eclipse/updates/3.8/R-3.8-201206081200/plugins/org.eclipse.equinox.console_1.0.0.v20120522-1841.jar

## Configure your portal-ext.properties

Open your portal-ext.properties and add the following:

```
extra.bundles.1=file:///the/path/where/you/put/libs
```
This is just a helper property so you can avoid the really long paths, but it's not required.

*Note*: The folder path must start with a protocol of `file:`

Next add this:

```
module.framework.initial.bundles=\
    felix-fileinstall.jar@start,\
    ${extra.bundles.1}/org.apache.felix.gogo.command-0.12.0.jar@start,\
    ${extra.bundles.1}/org.apache.felix.gogo.runtime-0.10.0.jar@start,\
    ${extra.bundles.1}/org.apache.felix.gogo.shell-0.10.0.jar@start,\
    ${extra.bundles.1}/org.eclipse.equinox.console_1.0.0.v20120522-1841.jar@start
```

That tells the Module Framework (our osgi) to start the four bundles/libs you downloaded. You can leave them where you dl'd them.

Now add the following:

```
module.framework.properties.osgi.console=11311
```

This is the port that the **Telnet Server** will bind to on your localhost. Note, we only want to bind to the `localhost` address for safety because this shell can do anything pretty much.
So, if you want to SSH to the shell, use the system's SSH and then just telnet to the localhost from there, it's safer for everyone.

## Accessing Gogo Shell over Telnet Server

So now you can start up the portal and if you can see the threads (say with Visual VM) you will note there is one named something like `equinox telnet`

Open up a command line, and type:

```
telnet localhost 11311
```

If everything went according to plan you should see something like this:

```
[rotty@rotty-xps ~]$ telnet localhost 11311
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
____________________________
Welcome to Apache Felix Gogo

g!
```

## Making it Usefull

So far so good? Ok! So what can you do with this?

There are a bunch of commands that are available out of the box. If you type

```
help
```

You will see output describing what commands are available and their function.
However that can be pretty useless without more conext of what this thing is all about.

## What are Bundles?
The first thing you want to understand is that `things` that get installed into OSGi are called `bundles`. **Bundles** are JAR files. They represent the basic unit of OSGi.

In order to see what bundles are installed you execute the command:

```
lb
```

This is the `List Bundles` command.

When I execute this command I see:

```
g! lb
START LEVEL 20
   ID|State      |Level|Name
    0|Active     |    0|OSGi System Bundle (3.8.0.v20120529-1548)
    1|Active     |    6|Apache Felix File Install (3.1.10)
    2|Active     |    6|Apache Felix Gogo Command (0.12.0)
    3|Active     |    6|Apache Felix Gogo Runtime (0.10.0)
    4|Active     |    6|Apache Felix Gogo Shell (0.10.0)
    5|Active     |    6|Console plug-in (1.0.0.v20120522-1841)
    6|Active     |    1|Gemini Management (1.0.1.RELEASE)
    7|Active     |    1|Apache Felix Declarative Services (1.6.2)
    8|Active     |    1|OSGi Release 4.2.0 Utility Classes (3.2.300.v20120522-1822)
    9|Active     |    1|osgi.enterprise (4.2.0.201003190513)
   10|Active     |    1|OSGi Release 4.2.0 Services (3.3.100.v20120522-1822)
```

## Installing from the Liferay SDK

Having access to the Gogo shell via Telnet is great but without knowing how to take part in installing bundles makes the framework as it stands not so usefull.

However, Liferay has simplified the deployment process of bundles for you to make it more familiar.

Luckily OSGi bundles are easily identifiable artifacts. Inside the JAR file is a mandatory `/META-INF/MANIFEST.MF` file.

When the Liferay Auto Deployer identifies such a jar, it simply places the jar file into the Module Framework Deploy folder. From there, the OSGi auto deploy mechansim loads the bundle and installs it.

As an example, if you have the Liferay Plugins SDK checked out from github enter the `<SDK>/shared/test-module-framework-shared/` directory.

Looking at the java source code of this plugin reveals a very simple class (removed whitespace and comments):

```java
package com.liferay.test.module.framework;
import aQute.bnd.annotation.component.Activate;
import aQute.bnd.annotation.component.Component;
import aQute.bnd.annotation.component.Deactivate;
@Component
public class TestComponent {
    @Activate
    public void activate() {
        System.out.println("Activate TestComponent");
    }
    @Deactivate
    public void deactivate() {
        System.out.println("Deactivate TestComponent");
    }
}
```

now, execute the command

```
ant deploy
```

Within a few moments you should see the Liferay log report

```
13:40:43,539 INFO  [com.liferay.portal.kernel.deploy.auto.AutoDeployScanner][AutoDeployDir:204] Processing test-module-framework-shared-7.0.0.1.jar
13:40:43,540 INFO  [com.liferay.portal.kernel.deploy.auto.AutoDeployScanner][ModuleAutoDeployListener:63] Copied module for /home/rotty/AS/liferay-portal/deploy/test-module-framework-shared-7.0.0.1.jar
13:40:43,541 INFO  [com.liferay.portal.kernel.deploy.auto.AutoDeployScanner][ModuleAutoDeployListener:69] Module for /home/rotty/AS/liferay-portal/deploy/test-module-framework-shared-7.0.0.1.jar copied successfully. Deployment will start in a few seconds.
Activate TestComponent
```

Note the last line which contains the same text as contained in the `activate()` method of the java class above.

Now, going back to the telnet session, re-execute the `lb` command.

```
g! lb
START LEVEL 20
   ID|State      |Level|Name
    0|Active     |    0|OSGi System Bundle (3.8.0.v20120529-1548)
    1|Active     |    6|Apache Felix File Install (3.1.10)
    2|Active     |    6|Apache Felix Gogo Command (0.12.0)
    3|Active     |    6|Apache Felix Gogo Runtime (0.10.0)
    4|Active     |    6|Apache Felix Gogo Shell (0.10.0)
    5|Active     |    6|Console plug-in (1.0.0.v20120522-1841)
    6|Active     |    1|Gemini Management (1.0.1.RELEASE)
    7|Active     |    1|Apache Felix Declarative Services (1.6.2)
    8|Active     |    1|OSGi Release 4.2.0 Utility Classes (3.2.300.v20120522-1822)
    9|Active     |    1|osgi.enterprise (4.2.0.201003190513)
   10|Active     |    1|OSGi Release 4.2.0 Services (3.3.100.v20120522-1822)
   13|Active     |    1|test-module-framework-shared (7.0.0.1)
g!
```

Notice the last line:

```
   13|Active     |    1|test-module-framework-shared (7.0.0.1)
```

This is the installed bundle. You can also see that it's `State` is `Active` which means it's already running so no problems existed with it.

Also notice that it has an `ID`. We need the id when performing operations on the bundle.

Now, let's stop the bundle.

```
g! stop "13"
Deactivate TestComponent
g!
```

Ah, notice that the text from the `deactivate()` command was printed. So, you have experienced the `install, resolve, start, stop` lifecycle of OSGi bundles.

An interesting point about the lifecycle states here is that currently the bundle is `stopped`.  and if you execute the `lb` command again you will see that the state of the bundle is in fact:

```
   13|Resolved   |    1|test-module-framework-shared (7.0.0.1)
```

Resolved means that it has no active components. If this bundle were to contain a fictional `portlet`, from this state it would not be registered in the portal.

Executing the `start` command will return the bundle to `Active` state if possible.

```
g! start "13"
Activate TestComponent
g!
```

Ok so we have a lifecycle. That's great if I want to start a Thread which does some operation until we tell it to stop, or the portal shuts down.

More to come...
