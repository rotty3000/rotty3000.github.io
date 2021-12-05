---
title: OSGi On Kubernetes - Tiny Java
excerpt: Significantly reduce the size of Java deployments by leveraging relatively simple OSGi and Java tooling.
---

There are a number of reasons why you should make every effort to reduce the size of your application deployments in Cloud infrastructures:

- to reduce number of bits being moved across network connections (too many people ignore this because "the images are cached")
- to lower the attack surface available to nefarious actors
- to reduce number of places where bugs can hide
- to improve startup time by reducing the number of resources to process at startup

There is, however, a rather troublesome issue with achieving this goal consistently and efficiently; **it's hard and very often painful!** Furthermore, there doesn't appear to be a particularly easy way to achieve it, no single recipe to help guide us.

Until now...

To clarify, there _are_ a number of frameworks/runtimes available today that directly target this concern. However, they are often _not real Java_ and require changing code or require an inordinate amount of proprietary details to set them up. I personally prefer to spend my time producing features using more traditional and standardized approaches instead of proprietary ones. These lend themselves far better to analysis or debugging using tools I'm already very comfortable with. The key of course is automation.

The goal of this post is to demonstrate a recipe which can be automated giving the smallest possible non-native Java images using common tools and technologies we already know and love (and maybe a couple that aren't widely used currently).

The first tool we'll talk about is my personal go-to for assembling OSGi applications and it has several incarnations for various build tool chains. It's called the Bnd. In this case I'm using the [`bnd-resolver-maven-plugin`](https://github.com/bndtools/bnd/blob/master/maven/bnd-resolver-maven-plugin/README.md) in (obviously) a Maven project. This is the same project I referred to in a [previous post]()