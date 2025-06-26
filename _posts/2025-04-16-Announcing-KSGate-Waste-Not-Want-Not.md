---
title: "Announcing KSGate - Waste Not, Want Not!"
excerpt: "Recently, there's a ton of energy spent in Kubernetes land thinking about how to save costs. Maybe KSGate can help!"
mermaid: true
tags: [kubernetes, ksgate, cel, resource-management]
date: 2025-06-25 19:00:00 -0500
---
> #### waste not, want not
> `PROVERB`<br/>
> if you use a commodity or resource carefully and without extravagance you will never be in need.

Recently, there's a ton of energy being spent in the Kubernetes ecosystem thinking about how to save costs.
[KSGate](https://github.com/ksgate/ksgate) is a prototype I've been working on to provide a generalized and declarative solution to scheduling.
One big side effect of strict scheduling is the reduction of costly resources by avoiding two aspects of 
Kubernetes that really tend to annoy me; wasteful scheduling and weak service dependencies.

Let me start by talking about these two things and then I'll explain how my experiment ([KSGate](https://github.com/ksgate/ksgate)) plays a role in addressing them.

We'll start with...

### Wasteful Scheduling

Consider the following. Imagine we have a database and a secret the database depends on.

```yaml
{% include_relative ksgate-announce/database-pod.yaml %}
---
{% include_relative ksgate-announce/database-secret.yaml %}
```

If you deploy the database but the secret does not exist the pod will be scheduled and the kubelet will try to get the secret. If the secret is not ready, the kubelet will mark the pod status as `CreateContainerConfigError` and try over and over to see if the secret ever does show up. So the resources for the pod are allocated on the node but the kubelet of the node can't actually run it. The kubelet is spending time and resources trying to get the secret none of which is useful to your application. And who knows when the secret will be ready?

This is just one example of wasteful scheduling but I hope you get the idea that wasteful scheduling is workload scheduling that doesn't provide any application benefit yet still incurs infrastructure costs in an uncontrolled fashion. Do you even measure how much cost is spent on compute that is not application related?

Let's move on to...

### Weak Dependencies

Now what if one resource depends on another resource and the other resource is not ready?

Consider the following. Here we have an application that depends on a database.

```yaml
{% include_relative ksgate-announce/app-pod.yaml %}
```

How might we model this in Kubernetes?

The first suggestion that you're likely to come across is to use an init container (or maybe a sidecar container) to wait for the database to be ready. Something like the following:

```yaml
{% include_relative ksgate-announce/app-pod-with-wait.yaml %}
```

While functional this approach is inefficient and wasteful because the application pod is scheduled and consuming resources even though it can't actually do something meaningful until the database is ready and there is no way to express this dependency in a way that the scheduler will respect it.

That's why I refer to this as a _weak dependency_.

Either way...

### Why should we care?

The two cases above have to be accepted as known wastes of resources. We know it will happen, but we can't really do anything about it. However, factored across the entire architecture this could amount to a significant cost.

Until recently, however, there were no better options. The scheduler is allocating resources for essentially useless processes that are consuming resources and executing logic which is not part of our application.

This means that we're not using our resources efficiently and as cloud resources increase in price this wastefulness saps the value from our infrastructure.

Enter...

### Scheduling Gates

Luckily the Kubernetes maintainers have thought about this and have put hooks in place around the scheduler which allow us to control the scheduling of workloads until some set of conditions are met. This mechanism is called _scheduling gates_ or more precisely [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/). It reached `stable` status in Kubernetes 1.30.

Today, we can create a scheduling gate for the database in the first scenario that indicates the need for the secret and then _wait for the gate to be removed_ before the database will be scheduled with zero application resources wasted.

```yaml
{% include_relative ksgate-announce/database-pod-with-gate.yaml %}
```

Since the scheduler won't schedule the database pod until the gate is removed no resources are wasted. There is no logic executing, no resources are even allocated let alone consumed, and this state can be maintained indefinitely at essentially zero cost.

Similarly, we can create a scheduling gate for the application that indicates the need for the database and again _wait for the gate to be removed_ before the application will be scheduled.

```yaml
{% include_relative ksgate-announce/app-pod-with-gate.yaml %}
```

### But Who's the Gatekeeper?

The above approach is a good start, but it has a few drawbacks. You'll note that I emphasized the phrase "_wait for the gate to be removed_". That harkens to the fact that no one currently recognizes the scheduling gate I used so I need to write a custom controller that recognizes and knows under what conditions to remove it. As I add more and more gates I need more controllers or more code to recognize the conditions implied by each gate. This sounds like a lot of work, maybe more work than what we had before.

#### Declarative Scheduling Gates

What if we could define a scheduling gate in a declarative way? What if there was a scheduling gate controller that could manage the scheduling gate for us, or even better, a scheduling gate controller that could manage the scheduling gate for us **and** allowed us to define the conditions under which the gate should be removed in a flexible way?

That sounds pretty interesting doesn't it?

Let's start by considering the constraints.

First thing to consider is that scheduling gates are just opaque strings with a maximum length of 63 characters and the characters allowed are limited to alphanumeric, `/`, `-`, and `_`. This means that we can't really use them for anything other than unique identifiers. 

On the other hand the constraints we want to declare are details that are closely tied to the definition of the workload so it shouldn't be kept far away from it or there's a risk of it getting out of sync. But we can't store that information in the scheduling gate so we need some other way to store it. It would be nice if we could avoid having to create a custom storage layer or a new CRD and force everyone to have to adapt to this new mechanism. That would make a generalized solution that no-one would adopt because it doesn't integrate into existing resource definitions (like Helm charts and such.)

If only there was a way to store arbitrary information in the workload definition itself that could be tied to the scheduling gate identifier without the hassle of new storage layer or new CRD.

#### Annotations to the Rescue

As it turns out, there is a way to do this. We could use the `metadata.annotations` on the workload to store the condition information and bind it to the scheduling gate identifier.

```yaml
{% include_relative ksgate-announce/app-pod-with-gate-anno.yaml %}
```

This approach of pairing a scheduling gate with a matching annotation, along with a little bit of _semantic-fu_ would allow us to declare conditions directly in the definition that a generic controller could use to determine when the workload is ready to be scheduled and do it in a way that is fully compatible with existing workload definitions.

## [KSGate](https://github.com/ksgate/ksgate)

I've modelled this idea and I'm working on a [controller](https://github.com/ksgate/ksgate) that will manage the _scheduling gate and annotation pairs_ for us. I've called this controller **KSGate** (*derived from **Kubernetes Scheduling Gates***).

In order to avoid conflicts with any pre-existing scheduling gates out in the wild, **KSGate** is designed to only consider gates that are prefixed with `k8s.ksgate.org/`. The suffix that must follow is arbitrary and is used by developers to uniquely identify managed gates on the workload (that gives 46 remaining characters to work with, but considering the scope of uniqueness is constrained to the resource itself that should leave plenty of room to work with).

### Conditions

Here's an example of a condition that defines a dependency on the existence of a secret named `db-secret`:

```json
{
  "apiVersion": "v1", // (optional) defaults to "v1", as in core/v1
  "kind": "Secret", // required
  "name": "db-secret", // required
  "namespace": "default", // (optional) defaults to the resources's namespace
}
```

Conditions specify the `apiVersion`, `kind`, and `name` to identify the resource that the condition is referencing. The `namespace` property is used to specify the namespace of the resource. When omitted, the pod's namespace is used. To target cluster scoped resources specify the `"namespaced": false` property.

_The schema for the annotation value of a condition can be found [here](https://github.com/ksgate/ksgate/blob/main/condition.schema.json)._

### How Conditions Work

The KSGate controller identifies any workloads that are `schedulingGated` and identifies any matching gates. For each matching gate found a watcher is created that will receive any events for the referenced resource and when the condition is met the gate is removed.

### Expressions

**Expressions** are what give real power to conditions.

The `expression` property on a condition is used to specify a [CEL](https://kubernetes.io/docs/reference/using-api/cel/) expression that must evaluate to true for the condition to be satisfied.

Here's an example of a condition that uses an expression to define a dependency on the Pod named `database` being in the `Running` phase:

```json
{
  "apiVersion": "v1", // (optional) defaults to "v1", as in core/v1
  "kind": "Pod", // required
  "name": "database", // required
  "namespace": "default", // (optional) defaults to the resources's namespace
  "expression": "resource.status.phase == 'Running'" // (optional) if omitted, the existence of the resource is used to satisfy the condition
}
```

The CEL environment contains two variables; `resource` and `this`:

- `resource` - gives access to the resource matched by the `apiVersion`, `kind`, and `name` properties
- `this` - gives access to the pod that is being scheduled.

A single function also extends the standard set of functions:

- `now()` - gives access to the current time so temporal expressions can be used. (_e.g. `(now() - timestamp(resource.metadata.creationTimestamp)) > duration('10s')` means that the resources was created at least 10 seconds before now_)

**CEL** expressions allow for complex conditions to be specified over the entire content of the pod (using `this`) and the target `resource`.

### Example

Let's see our original example with **KSGate**. You'll recall that we had a database workload and a secret that the database workload depended on.

```yaml
{% include_relative ksgate-announce/database-pod-with-ksgate.yaml %}
```

Now the database pod has a scheduling gate for the secret and it won't be scheduled until the secret exists, perhaps indefinitely and without any resource waste but will immediately be scheduled when the secret arrives.

Suppose we include the application such that it depends both on the secret and the database.

```yaml
{% include_relative ksgate-announce/app-pod-with-ksgate.yaml %}
```

Pretty neat, right?

### Closing thoughts

**KSGate** is a controller that enables you to define scheduling gates and the conditions under which they should be removed in a declarative way using powerful CEL expressions. In doing so it allows you to define scalable, indefinite, resource free and failure free dependencies. It can also reduce those wasteful scheduling scenarios and hopefully reduce costs.

I'm excited to see where this goes and I'm looking forward to seeing what other people think about this idea. To that end, I've enabled [GitHub Discussions](https://github.com/ksgate/ksgate/discussions) for the project where you can give feedback and share your thoughts and ideas. If you're interested in contributing to the project, please feel free to reach out to me there.

**KSGate** is licensed under the permissible **[Apache License 2.0](https://github.com/ksgate/ksgate/blob/main/LICENSE)**.

If you care to leave feedback, please do so on [LinkedIn](https://www.linkedin.com/in/raymond-auge/).