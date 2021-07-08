---
title: OSGi On Kubernetes - Tiny Java
excerpt: Significantly reduce the size of Java deployments by leveraging relatively simple OSGi and Java tooling.
---

There are a number of reasons why you should make every effort to reduce the size of your application deployments in Cloud infrastructures:

- to reduce number of bits being move across network connections
- to lower the attack surface available to nefarious actors
- to eliminate potential bugs or at least reduce number of paths when investigating bugs by keeping a clean _house_
- to improve startup time by reducing the number of resources scanned/loaded/introspected at startup

There is, however, a rather troublesome issue with achieving this goal consistently and efficiently; **it's hard and very often painful!** Furthermore, there doesn't appear to be a particularly easy way to achieve it, no single recipe to help guide us.

Until now...

To clarify, there _are_ a number of tools and frameworks available today that directly target this concern, however, they are often _not real Java_ and require changing code, or require an inordinate amount of proprietary details in setting them up. I personally prefer to spend my time producing features using more standardized avenues instead of doing such deep house keeping. If only it could be automated...

Before we dig in, however, I would like to contrast two different software assembly models; **_software distribution_** vs. **_fit-for-purpose software_**.

- **_software distribution_** - I define to be any collection of software and libraries which are pre-assembled without a specific intended use of the whole.

- **_fit-for-purpose software_** - I define as including exactly what is required for a given usage, and nothing more.

One might say that a _software distribution_ is synonymous with the word _server_ in the term _server-less_ where the _server_ has many more features than any particular use of said server needs in any given deployment and the _less_ means "_less_ the unused parts". You _might_ also wish to call a _software distribution_ a monolith. However I am remiss use the term monolith without including in the definition the aspect of being _unbreakable_ and _software distributions_ are very often _breakable_ so in my opinion the nomenclature doesn't fit.

The focus of Cloud applications today leans much more toward _fit-for-purpose software_ deployment that it does toward _software distribution_ deployment. Quite often the term _micro-service_ is thrown around to refer to this but I think it is often miss-used so I'll try to refrain from using it here.

So our goal today is to develop _fit-for-purpose software_ using the tools and technologies we know and love. The first tool we'll talk about is my corner stone for building OSGi assemblies and it has several incarnations for various build tool chains. It's called the Bnd Resolver. In this case I'm using the [`bnd-resolver-maven-plugin`](https://github.com/bndtools/bnd/blob/master/maven/bnd-resolver-maven-plugin/README.md) incarnation in a Maven project. This is the same project I referred to in a [previous post]()