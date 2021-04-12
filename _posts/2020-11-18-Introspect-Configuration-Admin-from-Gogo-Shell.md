---
title: Introspect Configuration Admin from Gogo Shell
excerpt: Apache Felix Gogo is by far one of the most potent tools in any OSGi development toolkit. And if you are fortunate enough to have access to a very modern version of Apache Felix Configuration Admin you can already benefit from the fact that you can interact with Configuration Admin through the gogo shell with the commands.
---

[Apache Felix Gogo](https://felix.apache.org/documentation/subprojects/apache-felix-gogo.html) is by far one of the most potent tools in any OSGi development toolkit. And if you are fortunate enough to have access to a very modern version of [Apache Felix Configuration Admin](https://github.com/apache/felix-dev/tree/master/configadmin) you can already benefit from the fact that you can interact with Configuration Admin through the Gogo shell with the commands:

* `cm:createfactoryconfiguration` - create a factory configuration (maps to the Configuration Admin method [`createFactoryConfiguration(String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#createFactoryConfiguration-java.lang.String-) or [`createFactoryConfiguration(String,String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#createFactoryConfiguration-java.lang.String-java.lang.String-))
* `cm:getconfiguration` - create a single configuration (maps to the Configuration Admin method [`getConfiguration(String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#getConfiguration-java.lang.String-) or [`getConfiguration(String,String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#getConfiguration-java.lang.String-java.lang.String-))
* `cm:getfactoryconfiguration` - get or create a factory configuration (maps to the Configuration Admin method [`getFactoryConfiguration(String,String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#getFactoryConfiguration-java.lang.String-java.lang.String-) or [`getFactoryConfiguration(String,String,String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#getFactoryConfiguration-java.lang.String-java.lang.String-java.lang.String-))
* `cm:listconfigurations` - list configurations (maps to the Configuration Admin method [`listConfigurations(String)`](https://docs.osgi.org/javadoc/osgi.cmpn/7.0.0/org/osgi/service/cm/ConfigurationAdmin.html#listConfigurations-java.lang.String-))

***Note:*** *You may find that when using `help` the commands come up in camel case. Luckily commands are case insensitive so you can use all lower case to simplify execution if you choose.*

### listconfigurations

The most useful function right off the bat is to list any existing configurations that might already exist in the framework using the `listconfigurations`:

```bash
g! listconfigurations "(service.pid=*)"
Configuration PID=foo.bar, factoryPID=null, bundleLocation=?
Configuration PID=foo.baz, factoryPID=null, bundleLocation=?
```

To get an individual configuration from the list you can use the array syntax:

```bash
g! c = (listconfigurations "(service.pid=?)") 0
Properties           [foo=bar, service.pid=foo.bar]
Attributes           []
FactoryPid           null
Pid                  foo.bar
ChangeCount          2
BundleLocation       ?

```

To list the properties of a configuration in tabular form you can use:

```bash
g! $c properties
foo                 bar
service.pid         foo.bar

```

### getconfiguration: Get or create single configurations

You can get a configuration using:

```bash
g! c = getconfiguration "foo.biz"
Properties           null
Attributes           []
FactoryPid           null
Pid                  foo.biz
ChangeCount          1
BundleLocation       reference:file:???????/org.apache.felix.gogo.runtime-1.1.4.jar

```

**However**, you'll note that if the configuration doesn't exist a new instance is returned. The problem is that if you use the single argument `getconfiguration` AND the configuration does not already exist what you will get is an instance that is only visible to the bundle that triggered the creation (notice the `BundleLocation` above), which is not usually what you want. In that case you should use:

```bash
g! c = getconfiguration "foo.bom" "?"
Properties           null
Attributes           []
FactoryPid           null
Pid                  foo.bom
ChangeCount          1
BundleLocation       ?

```

**Note** that now you have a `BundleLocation` of `?` which means it is visible to any bundles that want to read it.

If you don't want the *or create* behaviour it's best to use `listconfigurations` when searching for configurations (it's more powerful anyway since you can filter on any properties of the configurations).

### Create configurations from scratch

So let's say that no configuration exist for the PID and you also want to configure it with properties. First you need a way to create a `Dictionary` object since that is the argument required by the `Configuration.update(Dictionary)` method.

You can use the following to create a dictionary object:

```bash
g! d = (((bundle 0) loadclass java.util.Hashtable) newInstance)

```
Add some properties:
```bash
g! $d put "Foo" "bar"
```
Print the result:
```bash
g! $d
Foo                 bar

```

Now you can pass this dictionary to the configuration you have stored in the var `c`:

```bash
g! $c update $d
g! $c
Properties           [Foo=bar, service.pid=foo.bom]
Attributes           []
FactoryPid           null
Pid                  foo.bom
ChangeCount          2
BundleLocation       ?

```

### createfactoryconfiguration: Create factory configurations

The basic incarnation of `createfactoryconfiguration` has the same gotchas as `getconfiguration` where it will create a configuration with a `BundleLocation` limited to the caller bundle. So you should prefer the two argument version as follows:

```bash
g! c = createfactoryconfiguration "foo.biz" "?"
Properties           null
Attributes           []
FactoryPid           foo.biz
Pid                  foo.biz.54bf0c1f-a7bd-4bfd-a80f-fedf58793442
ChangeCount          1
BundleLocation       ?

```

Notice that now the PID contains a generated part, e.g. `foo.biz.54bf0c1f-a7bd-4bfd-a80f-fedf58793442`.

Setting configuration properties is exactly the same as above.

### getfactoryconfiguration: Named factory configurations

Recent versions of Configuration Admin specification make it possible to create *named* configurations. The command `getfactoryconfiguration` follows the pattern `getConfiguration` with it's *get or create* paradigm. However, the difference between this and the `createfactoryconfiguration` is the *name* argument which allows passing a human readable string to uniquely disambiguate factory configurations beyond the opaque generated suffix normally used for factory configurations which make them hard to identify. Here we also prefer adding the third argument so that we make the configuration readable by any bundle.

```bash
g! c = getfactoryconfiguration "foo.biz" "one" "?"
Properties           null
Attributes           []
FactoryPid           foo.biz
Pid                  foo.biz~one
ChangeCount          1
BundleLocation       ?

```

Notice that in this case the PID is suffixed with a tilde (`~`) followed by the human readable string argument we passed (`one`).

### What about older versions of Felix Configuration Admin

Earlier I mentioned:

> if you are fortunate enough to have access to a very modern version of [Apache Felix Configuration Admin](https://github.com/apache/felix-dev/tree/master/configadmin) you can already benefit from the fact that you can interact with Configuration Admin through the Gogo shell with the commands

Well, it turns out that even if you are using an older version you can still do most of the above provided you create a command from the Configuration Admin service (if the service is available).

First you may need to setup the command context so that you have a few basic commans:

```bash
g! addcommand context ${.context}
```
*(This gives you access to commands like `servicereference <fqcn|filter>`.)*

Now you can add the config admin service commands with:

```bash
g! addcommand cm (service (servicereference "org.osgi.service.cm.ConfigurationAdmin"))
```

That's it! Now everything above should work exactly as documented.