---
title: Breathing new life into JSP with OSGi!
excerpt: Presentation at Eclipsecon Europe 2014
---

## Outline

* JSP!!! What's the meaning of this?
* Bytecode is Better
* Initial Problems
* Compiling
* Designing a Solution
* Key Java EE OSGi Bundles
* Jasper*
* Compilation - javax.tools
* OSGi Support - org.osgi.framework.wiring
* Phidias
* Liferay JSP Compiler
* Some Details
* Demo Time

### JSP!!! What's the meaning of this?

* Isn't *JSP* a dead tech?
* Enterprises use *JSP* *a lot*
  * `find -type f -regex .*\.jspf? | wc -l`
    * Eclipse = 59
    * Liferay = 1696
    * Liferay Plugins = 669

### Bytecode is Better

* *Compiled to Bytecode* view templates _still_ most efficient server side
  * w.r.t. memory consumption and performance
  * Other *_compiled to bytecode_* view tech not as widely known
    * *Play Scala Templates*
    * *Google Soy Templates*

### Initial Problems

* Java EE instantiation model (`ServiceLoader`, `Class.forName`, TCCL)
* Limitations to the existing JSP servlet impls required ugly hacks
* Compilation dependencies lead to complex runtime dependencies

### Compiling

* Traditionally - JDT *_load the world_* compilation strategy
  * JDT is not a small dependency
* Other compilation options non-existent
* OSGi had no way to natively support compilation process

### Designing a Solution

* Re-use existing tech when possible
* Use JDK compiler API (since Java 6)
* Avoid lots of hacks
* Keep a small footprint
* Limit usage requirements

### Key Java EE OSGi Bundles

* Key API/RIs are OSGi bundles
** Servlet
** JSP
** JSTL
** EL

### Jasper

* Proven and stable
* Has limited dependencies
* Recently solved issues enabling easier OSGi use

Thanks Glassfish!

### Java SE Support - javax.tools

* Since Java SE 6
* Full blown compiler support
* .. complicated to use

### OSGi Support - org.osgi.framework.wiring

* Since Core R4.3
* Deep classloader & wiring resource access

### Phidias

* Small OSS bundle which crosses
  * `javax.tools`
  * `org.osgi.framework.wiring`
* Provides simplified OSGi aware compiler support
* Alpha (0.3.4)

### Liferay JSP Compiler

* Extends Jasper with custom
  * JSP compiler which uses Phidias
  * JSP servlet which transparently handles classloading concerns

### Some Details

* Entire JSP support ~ 1.3MB
* Most modern versions of JSP, JSTL, EL available
* No need to declare any JSP related imports in JSP _enabled_ bundles
* No need to include any libs or standard TLDs
* No need to WAB your bundles
  * Can use regular old jar layout + web fragment subtree
    * Place JSPs in `META-INF/resources/`

### OSGi Aware Compilation

* Does this open the door for other interesting topics?
  * Limit compile time view of the world to public APIs?
