---
title: "Using Requirements and Capabilities"
excerpt: "The Requirements and Capabilities model is a surprisingly powerful concept born from the 17 years of experience the OSGi Alliance has in designing modular software: defining strong contracts."
---

I really love how everyone at Liferay is taking to the OSGi development model. It makes me proud to see how much work has been done in this regard.
There's one very cool area I think is worth expanding on.

## Requirements and Capabilities

What is it? Some history.

The Requirements and Capabilities model is a *surprisingly powerful* concept born from the 17 years of experience the OSGi Alliance has in designing modular software: defining *strong contracts*. In OSGi contracts are described with meta-data and enforced by a strict runtime.

The process involved in defining these contracts led to a frequently repeating pattern. First a contract was defined using special manifest headers and their specific semantics. An example of this is the `Import-Package` header which is used to specify what java packages should be made available so that your code can execute. This was followed by the need to implement the specific logic to enforce the definition. The result of this work manifested itself (excuse the pun) by what we recognize as OSGi Bundle manifest headers and the OSGi frameworks that enforce the semantics of those headers.

Around 2005 some very smart people in the OSGi community and OSGi Alliance recognized a pattern and developed it into a *constraint model* called *Requirements and Capabilities*.

### How does it work?

A contract is defined starting with a unique namespace. Within this namespace semantics are defined for a set of *attributes* and *directives*. The entire definition forms a type of language from which can be created instances of *Capabilities* and *Requirements*.

Let's take an example.

Suppose I want to describe a service (which begins by defining a contract) where people can take their pets to be groomed. There are many types of pets, and many grooming agencies who can only groom certain kinds of pets because of the special skills, equipment or facilities required for each type. It can be a challenge to find an appropriate agency to groom your pet.

Let's imagine that every agency declared their capabilities using a single *name-space* called `pet.grooming` having 4 *attributes*:
* **type** - *a list of strings* naming the type of pets groomed by the agency
* **length** - *a positive integer* specifying the maximum size of the pet which the agency can groom
* **soap** - *a string* naming the type of soap used by the agency
* **rate** - *a positive integer* specifying the rate per hour charged by the agency

Here we have three example agencies using this contract in the syntax found within an OSGi Bundle manifest:

* Agency A: Haute Pet Coiffure
```
Provide-Capability: pet.grooming;type:List="dog,cat";length:Long=800;soap="organic";rate:Long="50"
```

* Agency B: Great Big Pets
```
Provide-Capability: pet.grooming;type:List="cat,horse";length:Long=3000;soap="commercial";rate:Long="20"
```

* Agency C: Joe's Pets
```
Provide-Capability: pet.grooming;type:List="dog,cat";length:Long=1500;soap="regular";rate:Long="15"
```

Clients could then declare their *Requirements* using the `pet.grooming` *name-space* and a special *LDAP filter*.

Let's take a look at 6 clients:

* Client A:
```
Require-Capability: pet.grooming;filter:="(&(type=cat)(rate&lt;=20))"
```

  Which agencies do you think satisfy this requirement?

  *(hint: B & C)*

* Client B:
```
Require-Capability: pet.grooming;filter:="(&(type=dog)(length&gt;=1000))"
```

  Which agencies do you think satisfy this requirement?

  *(hint: C)*

* Client C:
```
Require-Capability: pet.grooming;filter:="(type=horse)"
```

  Which agencies do you think satisfy this requirement?

  *(hint: B)*

* Client D:
```
Require-Capability: pet.grooming;filter:="(&(type=dog)(soap=organic))"
```

  Which agencies do you think satisfy this requirement?

  *(hint: A)*

* Client E:
```
Require-Capability: pet.grooming;filter:="(type=cat)"
```

  Which agencies do you think satisfy this requirement?

  *(hint: A & B & C)*

* Client F:
```
Require-Capability: pet.grooming;filter:="(type=dragon)"
```

  Which agencies do you think satisfy this requirement?

  *(hint: ???)*

**Observation:** What happens for client *F*? This is a case where requirements cannot be satisfied. What does this mean? This might translate directly into the resource containing this requirement not *resolving*. In other words, it might be completely blocked from doing whatever it was it intended to do. This is a remarkable characteristic. That we could know in a safe and reproducible way that a resource cannot be satisfied could prevent any number of catastrophic situations we would hard pressed recovering from at runtime.

Once again, note the *name-space* and the *filter* used to query or match the attributes of the available *Capabilities*.

This language which first materialized as **OSGi RFC 112** is very powerful and can model a wide range of contracts. It's power was demonstrated by the fact that all bundle headers from prior OSGi specifications could be modelled by it. It also became possible to implement an engine which could calculate a set of resources given an initial set of requirements. This engine is known as *the resolver* and all OSGi frameworks beginning at release R4.3 have such a resolver at their heart. Since then it's possible by specifying new name-spaces to model new contracts specific to your own needs. These new contracts play at par with any of the OSGi defined name-spaces.

In Liferay we have used this language to generalize contracts for *JSP Tag Libraries* to enable modularity around their use, for *Service Builder* extender support, so that the correct version of service builder framework is available to support your SB modules. We have also used it to create a prototype of [RFP 171 - Web Resources](https://github.com/osgi/design/blob/master/rfps/rfp-0171-Web-Resources.pdf) to enable modular UI development.

One of great benefits of having such succinct way of defining a contract is that much of the information can be auto generated through introspection which makes it both easy to implement and use. The majority of cases require little to no effort from the developers who are requiring capabilities, and greatly reduces the effort for developers who are providing them.

Recently the [bnd](https://github.com/bndtools/bnd) project was enhanced with a set of annotations to easily produce arbitrary Requirements and Capabilities automatically. These annotations can be seen in use in the [OSGi enRoute](http://enroute.osgi.org/) project.

As a follow up to the bnd POC an RFP was submitted to and accepted by the OSGi Alliance under [RFP 167 - Manifest Annotations](https://github.com/osgi/design/blob/master/rfps/rfp-0167-Manifest-Annotations.pdf) to specify a standard set of annotations for simplifying and enhancing manifest management programmatically, including **Requirements and Capabilities**.

There is a lot of exciting work going on in this area and many opportunities to get involved, the simplest of which is giving feedback on or testing current work.

How could you use the **Requirements and Capabilities** model in your project?
