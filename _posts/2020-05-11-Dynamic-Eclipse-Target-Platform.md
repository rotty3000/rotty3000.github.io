---
title: Dynamic Eclipse Target Platform
excerpt: I've been one of the maintainers of the Eclipse Equinox Http Servlet bundles for several years now.
---

I've been one of the maintainers of the Eclipse Equinox Http Servlet bundles for several years now. I don't work on many Equinox implementations so this task is not part of my daily grind and I quite often forget how to get back into it when needed.

The Eclipse Platform is now releasing quite frequently and this is awesome in many respects, but one side effect is that the configuration in Eclipse needed to work against the HEAD of master is constantly changing and managing this, or figuring out what is required to get things working in the IDE again can be a pain.

By far the most annoying detail is the Target Platform and specifically a couple of Orbit bundles I need for equinox http servlet. The issue is caused by the version of the Orbit drop and Platform SDK always changing for obvious reasons.

The solution however came in the form of a tutorial on [vogella.com](https://www.vogella.com/) called [Eclipse Target Platform](https://www.vogella.com/tutorials/EclipseTargetPlatform/article.html).

The interesting details are about the dependency versions. Apparently in editing the `.target` resource you can use the placeholder version `0.0.0` to indicate using the latest.

The other interesting detail is that the 2 P2 sites I care about for this target platform both have _latest_ aliases:

* [http://download.eclipse.org/releases/latest](http://download.eclipse.org/releases/latest)
* [https://download.eclipse.org/tools/orbit/downloads/latest-R/](https://download.eclipse.org/tools/orbit/downloads/latest-R/)

I don't know about you but I find these aliases to be notoriously hard to find.

So now I have an eclipse project in my workspace called `target-platform` like in the *Vogella* tutorial and a `latest-target-platform.target` file which contains:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?pde version="3.8"?>
<target name="latest-target-platform">
	<locations>
		<location includeAllPlatforms="false" includeConfigurePhase="true" includeMode="planner" includeSource="true" type="InstallableUnit">
			<repository location="http://download.eclipse.org/releases/latest"/>
			<unit id="org.eclipse.sdk.feature.group" version="0.0.0"/>
		</location>
		<location includeAllPlatforms="false" includeConfigurePhase="true" includeMode="planner" includeSource="true" type="InstallableUnit">
			<repository location="https://download.eclipse.org/tools/orbit/downloads/latest-R/"/>
			<unit id="org.apache.commons.fileupload" version="0.0.0"/>
			<unit id="org.apache.commons.fileupload.source" version="0.0.0"/>
		</location>
	</locations>
</target>
```

Now I hope I never have to search for the correct Orbit and Eclispe SDKs ever again.