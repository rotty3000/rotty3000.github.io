---
title: "Announcing Gateman - Waste Not, Want Not!"
excerpt: "Recently, Kubernetes has me thinking about how to avoid wastefulness in the scheduling of workloads. Gateman to the rescue!"
mermaid: true
tags: [kubernetes, gateman, cel]
date: 2025-03-02 19:00:00 -0500
---
> #### waste not, want not
> `PROVERB`<br/>
> if you use a commodity or resource carefully and without extravagance you will never be in need.

Recently, Kubernetes has me thinking about how to save costly resources by avoiding two aspects of its architecture; wasteful scheduling and weak service dependencies.

Let's start with...

### Wasteful Scheduling

Consider the following. Imagine we have a database and a secret the database depends on.

```yaml
{% include_relative gateman-announce/database-pod.yaml %}
---
{% include_relative gateman-announce/database-secret.yaml %}
```

If you deploy the database but the secret does not exist the pod will be scheduled and the kubelet will try to get the secret. If the secret is not ready, the kubelet will mark the pod status as `CreateContainerConfigError` and try over and over to see if the secret ever does show up. So the resources for the pod are allocated on the node but the kubelet of the node can't actually run it, it's spending time and resources trying to get the secret neither of which are what your application is interested in. And who knows when the secret will be ready?

This is just one example of wasteful scheduling.

### Weak Service Dependencies

Now what if the pod depends on another pod and the other pod is not ready?

Consider the following. Here we have an application that depends on the database.

```yaml
{% include_relative gateman-announce/app-pod.yaml %}
```

How might we model this in Kubernetes?

The first suggestion that you're likely to come across is to use an init container (or maybe a sidecar container) to wait for the database to be ready. Something like the following:

```yaml
{% include_relative gateman-announce/app-pod-with-wait.yaml %}
```

This approach is inefficient and wasteful because the application pod is consuming resources even though it can't actually do something meaningful until the database is ready and there is no way to express this dependency in a way that the scheduler will respect it. 

That's why this is a _weak service dependency_.

### Why should we care?

The two scenarios above amount to a considerable waste of resources. I don't have concrete numbers but anecdotal evidence from a busy cluster leads me to believe that this not an insignificant cost. However, until recently, there were no better options. The scheduler is spending time allocating resources for essentially useless processes that are consuming resources and executing logic which is not part of our application.

This means that we're not using our resources efficiently and as cloud resources increase in price this wastefulness saps the value of our infrastructure.

### Scheduling Gates

Luckily the Kubernetes maintainers have thought about this and have put hooks in place around the scheduler which allow us to control the scheduling of workload pods until some set of conditions are met. This mechanism is called _scheduling gates_ or [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/). It reached `stable` status in Kubernetes 1.30.

Today, we can create a scheduling gate for the database in the first scenario that indicates the need for the secret and then _wait for the gate to be removed_ before the database will be scheduled.

```yaml
{% include_relative gateman-announce/database-pod-with-gate.yaml %}
```

Since the scheduler won't schedule the database pod until the gate is removed no resources are wasted. There is no logic executing, no resources are consumed and this state can be maintained indefinitely at essentially no cost.

Similarly, we can create a scheduling gate for the application that indicates the need for the database and then _wait for the gate to be removed_ before the application will be scheduled.

```yaml
{% include_relative gateman-announce/app-pod-with-gate.yaml %}
```

### But Who's the Gatekeeper?

The above approach is a good start, but it has a few drawbacks. You'll note that I italicized the phrase _wait for the gate to be removed_. That harkens back to the fact that no one currently recognizes the scheduling gate I used and so I need to write a custom controller that recognizes and knows under what conditions to remove it. As I add more and more gates I need more controllers or more code to recognize the conditions implied by each gate. This sounds like a lot of work.

#### Declarative Scheduling Gates

What if we could define a scheduling gate in a declarative way? What if there was a scheduling gate controller that could manage the scheduling gate for us, or even better, a scheduling gate controller that could manage the scheduling gate for us **and** allowed us to define the conditions under which the gate should be removed in a declarative way?

First thing to consider, however, is that scheduling gates are just opaque strings with a maximum length of 63 characters and the characters allowed are limited to alphanumeric, `/`, `-`, and `_`. This means that we can't really use them for anything other than a unique identifier. 

On the other hand dependency information is something that is closely tied to the definition of the workload so it shouldn't be kept far away from it or there's a risk of it being forgotten. But we can't store that information in the scheduling gate so we need some other way to store it.

If only there was a way to store arbitrary information in the workload definition that could be tied to the scheduling gate identifier.

#### Annotations to the Rescue

As it turns out, there is a way to do this. We could use the `metadata.annotations` on the workload to store the condition information and bind it to the scheduling gate identifier.

```yaml
{% include_relative gateman-announce/app-pod-with-gate-anno.yaml %}
```

This approach of pairing a scheduling gate with a matching annotation, along with a little bit of _semantic-fu_ would allow us to declare conditions directly in the definition that a generic controller could use to determine when the workload is ready to be scheduled.

## Gateman

I've modelled this idea and I'm working on a [controller](https://github.com/kdex-tech/kdex-gateman) that will manage the _scheduling gate and annotation pairs_ for us. I've called this controller **Gateman** because it's a gatekeeper for the scheduling gates.

In order to avoid conflicts with any pre-existing scheduling gates, **Gateman** is designed to only consider gates that are prefixed with `gateman.kdex.dev/`. The suffix is arbitrary and used by developers to uniquely identify the gate on the workload (that gives 46 remaining characters to work with, but considering the scope of uniqueness is constrained to the resource itself there should be plenty of room to work with).

### Conditions

Here's an example of a condition that defines a dependency on the existence of a secret named `db-secret`:

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "name": "db-secret",
  "namespace": "default" // optional, defaults to the resources's namespace
}
```

Conditions specify the `apiVersion`, `kind`, and `name` used to identify the resource that the condition is about. The `namespace` property is used to specify the namespace of the resource. When omitted, the pod's namespace is used.

_The schema for the annotation value of a condition can be found [here](https://github.com/kdex-tech/kdex-gateman/blob/main/condition.schema.json)._


### Expressions

Giving real power to conditions are the **expressions**.

Here's an example of a condition that uses an expression to define a dependency on the Pod named `database` being in the `Running` phase:

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "name": "database",
  "namespace": "default", // optional, defaults to the resources's namespace
  "expression": "resource.status.phase == 'Running'" // optional, if omitted, the existence of the resource is used to satisfy the condition
}
```

The `expression` property is used to specify a [CEL](https://kubernetes.io/docs/reference/using-api/cel/) expression that must evaluate to true for the condition to be satisfied.

The CEL environment contains two variables; `resource` and `pod`:

- `resource` - gives access to the resource matched by the `apiVersion`, `kind`, and `name` properties
- `pod` - gives access to the pod that is being scheduled.

CEL expressions allow for complex conditions to be specified over the entire content of the `pod` and `resource`.

### Example

Let's see our original example with **Gateman**. You'll recall that we had a database workload and a secret that the database workload depended on.

```yaml
{% include_relative gateman-announce/database-pod-with-gateman.yaml %}
```

Now the database pod has a scheduling gate for the secret and it won't be scheduled until the secret exists, perhaps indefinitely and without any resource waste.

Suppose we include the application such that it depends both on the secret and the database.

```yaml
{% include_relative gateman-announce/app-pod-with-gateman.yaml %}
```

Pretty neat, right?

### Closing thoughts

Gateman is a controller that enables you to define scheduling gates and the conditions under which they should be removed in a declarative way using powerful CEL expressions. In doing so it allows you to define scalable, indefinite, resource free and failure free dependencies.

I'm excited to see where this goes and I'm looking forward to seeing what other people think about this idea. To that end, I've enabled [GitHub Discussions](https://github.com/kdex-tech/kdex-gateman/discussions) for the project where you can give feedback and share your thoughts and ideas. If you're interested in contributing to the project, please feel free to reach out to me there.

If you care to leave feedback, please do so on [LinkedIn](https://www.linkedin.com/in/raymond-auge/).