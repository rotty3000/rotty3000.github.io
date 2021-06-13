---
title: Massive Enterprise Product Migration to OSGi
excerpt: Presentation from EclipseCon Europe 2015
---

Migrating from monoliths to microservices is not just a trend, it's a strategy for saner, happier living.

- what migration strategy do you use?
	Super bundle
	Complete re-write
	Embedded framework evolution

- services first!!!!

### Boostrapping embedded OSGi

```java
List<FrameworkFactory> frameworkFactories = ServiceLoader.load(
	FrameworkFactory.class);

FrameworkFactory frameworkFactory = frameworkFactories.get(0);

Map<String, String> properties = _buildFrameworkProperties(
	frameworkFactory.getClass());

_framework = frameworkFactory.newFramework(properties);

_framework.init();
```

### Lifecycle of a WAR

```java
public void contextInitialized(ServletContextEvent event) {
	try {
		_initFramework();

		super.contextInitialized(event);
	}
	finally {
		_startRuntime();
	}
}
```

Migrating from monoliths to microservices is not just a trend, it's a strategy for saner, happier living. This talk is one company's story about such a migration; the joys, the pains, and the outcome and how OSGi microservices helped.

### Restricting the view of the world

```java
private ClassLoader moduleFrameworkClassLoader() throws Exception {
	List<ClasspathResolver> classpathResolvers = ServiceLoader.load(
		ClasspathResolver.class);

	ClasspathResolver classpathResolver = classpathResolvers.get(0);

	return new ModuleFrameworkClassLoader(
		classpathResolver.getClasspathURLs(),
		ClassLoaderUtil.getPortalClassLoader());
}
```