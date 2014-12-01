---
layout: post
title: "Minimalist Build of Equinox Http From Source"
modified:
categories: blog
excerpt: "Building eclipse parts can be pretty hard. There are some great tutorials out there ..."
tags: [build, eclipse, equinox, "http servlet"]
image:
  feature:
date: 2014-12-01T17:28:03-05:00
author: raymond_auge
share: true
---

Building eclipse parts can be pretty hard. There are some great tutorials out there for building the entire platform project [Platform Build](https://wiki.eclipse.org/Platform-releng/Platform_Build), [Eclipse Target Platform - Tutorial](http://www.vogella.com/tutorials/EclipseTargetPlatform/article.html).

However, that's a huge ~25GB investment. Also, few examples demonstrate building just a subset of the projects in the platform.

As a new committer on only one of the multitude of eclipse platform projects I wanted an easier and simpler build path. This is particularly important when trying to recruite help from developers who might be interested in lending a hand but only in a very specific subset of eclipse platform.

In this case, I'm interested in building equinox `http.servlet` (and dependencies).

I need:

* the osgi core framework (`rt.equinox.framework/bundles/org.eclipse.osgi`)
* the osgi service bundle (`rt.equinox.framework/bundles/org.eclipse.osgi.services`)
* the http.servlet bundle (`rt.equinox.bundles/bundles/org.eclipse.equinox.http.servlet`)

I also want to make sure I can consume these from a maven repo, so I need to ensure the parent dependencies are setup correctly (i.e. make sure the parent poms are published as well).

Using the following commands I can do just that!

{% highlight bash %}
# Save this in your .profile or .bashrc for reuse, speeds up the build
export MAVEN_OPTS="-Xmx2048m -Declipse.p2.mirrors=false"

# Clone the platform aggregator repo and init the submodules
# Note: we deliberately didn't recurse through the modules
git clone -b master --progress git://git.eclipse.org/gitroot/platform/eclipse.platform.releng.aggregator.git
cd eclipse.platform.releng.aggregator
git submodule init

# since we want to push this into a maven repo we need the eclipse-platform-parent parent pom
cd eclipse-platform-parent/
mvn -N clean install

# get the framework repo
cd ..
git submodule update rt.equinox.framework/

# get the bundles repo
git submodule update rt.equinox.bundles/

# checkout master branch and install the rt.equinox.framework parent pom
cd rt.equinox.framework/
git co master
mvn -N clean install

# build the core framework
cd bundles/org.eclipse.osgi/
mvn -N clean install

# build the compendium services
cd ../org.eclipse.osgi.services/
mvn -N clean install

# now switch to the bundles repo and checkout master branch
cd ../../../rt.equinox.bundles/
git co master

# install the rt.equinox.bundles parent pom
mvn -N clean install

# finally build the http.servlet bundle
cd bundles/org.eclipse.equinox.http.servlet
mvn -N clean install
{% endhighlight %}

That's a mouthfull but once this is done just doing git updates and then rebuilding the individual bundles as needed is all you need to do.