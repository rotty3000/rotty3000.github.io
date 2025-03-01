---
title: Kubernetes - Waste Not, Want Not!
excerpt: Recently, Kubernetes has had me thinking about the wastefulness of workload reconciliation and workload inter-dependency.
mermaid: true
---
> #### waste not, want not
> `PROVERB`<br/>
> if you use a commodity or resource carefully and without extravagance you will never be in need.

Recently, Kubernetes has had me thinking about _the wastefulness of workload reconciliation_ and _workload inter-dependency_. Let's start with an explanation of what I mean by _the wastefulness of workload reconciliation_.

#### The Wastefulness of Workload Reconciliation

Consider the following. Imagine we have a database workload and a secret the database workload depends on.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
    - name: database
      image: postgres:16
      volumeMounts:
        - name: db-secret-volume
          readOnly: true
          mountPath: "/etc/db-secret"
  volumes:
    - name: db-secret-volume
      secret:
        secretName: db-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
data:
  username: ...
  password: ...
```

If you deploy the database workload but the secret does not exist the database workload will be scheduled and the kubelet will try to get the secret. If the secret is not ready, the kubelet will give up and the database workload will fail with the only recovery option being to delete the database workload and try again.

This is a waste of resources and can require manual intervention to recover.

---

Now let's talk about what I mean by _workload inter-dependency_.

#### Workload Inter-Dependency

Consider the following. Imagine we have an application workload that depends on a database workload.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  containers:
    - name: application
      image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
    - name: database
      image: postgres:16
```

How might we model this in Kubernetes?

The first suggestion that you'll come across in search engine results is to use an init container (or maybe a sidecar container) to wait for the database to be ready.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  initContainers:
    - name: wait-for-database
      image: busybox
      command: ["sh", "-c", "until nc -z database 5432; do echo waiting for database; sleep 1; done"]
  containers:
    - name: application
      image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
    - name: database
      image: postgres:16
```

The wait approach is inefficient.

#### Why should we care?

This approach of try-fail-retry is a huge waste of resources even though until recently it was the only option. The scheduler is spending time allocating resources for essentially useless processes that are consuming resources and executing logic which is not part of our application.

I couldn't imagine how many resources are wasted and how much time the scheduler spends handling these situations in a cluster of any size.

The wastefulness is a problem because it means that we're not using our resources efficiently and as cloud resources increase in price, this wastefulness becomes more and more costly and saps the value of our infrastructure.

#### Scheduling Gates

Luckily the Kubernetes maintainers have thought about this and have put hooks in place around the scheduler which allow us to control the scheduling of workload pods until some set of conditions are met. This mechanism is called _scheduling gates_ or [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/). It reached `stable` status in Kubernetes 1.30.

Today, we can create a scheduling gate for the database in the first scenario that indicates the need for the secret and then _wait for the gate to be removed_ before the database workload will be scheduled.

```yaml
# skip volumes
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
    - name: database
      image: postgres:16
  schedulingGates:
    - name: my.scheduling.gate/db-secret
```

Since the scheduler won't schedule the database pod until the gate is removed no resources are wasted. There is no logic to fail and this state can be maintained indefinitely at essentially no cost.

Similarly, we can create a scheduling gate for the application in the second scenario that indicates the need for the database and then _wait for the gate to be removed_ before the application will be scheduled.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  containers:
    - name: application
      image: nginx
  schedulingGates:
    - name: my.scheduling.gate/db-pod
---
# skip volumes
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
    - name: database
      image: postgres:16
  schedulingGates:
    - name: my.scheduling.gate/db-secret
```

Again, it prevents the application from being scheduled until the database is ready so it does not waste resources.

#### But Who's the Gatekeeper?

The above approach is a good start, but it has a few drawbacks. You'll note that I italicized the phrase _wait for the gate to be removed_. That harkens back to the fact that no one currently recognizes the scheduling gate I used and so I need to write a custom controller that recognizes and knows under what conditions to remove it. As I add more and more gates I need more controllers or more code to recognize the conditions implied by each gate. This sounds like a lot of work.

#### Declarative Scheduling Gates

What if we could define a scheduling gate in a declarative way? What if there was a scheduling gate controller that could manage the scheduling gate for us, or even better, a scheduling gate controller that could manage the scheduling gate for us **and** allowed us to define the conditions under which the gate should be removed in a declarative way?

First thing to consider, however, is that scheduling gates are just opaque strings with a maximum length of 63 characters and the characters allowed are limited to alphanumeric, `/`, `-`, and `_`. This means that we can't really use them for anything other than a unique identifier. 

On the other hand dependency information is something that is closely tied to the definition of the workload so it shouldn't be kept far away from it or there's a risk of it being forgotten. But we can't store that information in the scheduling gate so we need some other way to store it.

If only there was a way to store arbitrary information in the workload definition that could be tied to the scheduling gate identifier.

#### Annotations to the Rescue

As it turns out, there is a way to do this. We could use the `metadata.annotations` on the workload to store the condition information and bind it to the scheduling gate identifier.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
  annotations:
    my.scheduling.gate/database: "...condition..."
spec:
  containers:
    - name: application
      image: nginx
  schedulingGates:
    - name: my.scheduling.gate/database
```

This approach of pairing a scheduling gate with a matching annotation, along with a little bit of semantic fu would allow us to declare conditions directly in the workload definition that a generic controller could use to determine when the workload is ready to be scheduled.

#### Gateman

I've modelled this idea and I'm working on a [controller](https://github.com/kdex-tech/kdex-gateman) that will manage the _scheduling gate and annotation pairs_ for us. I've called this controller **Gateman** because it's a gatekeeper for the scheduling gates.

In order to avoid conflict with any pre-existing scheduling gates, **Gateman** is designed to only consider those that are prefixed with `gateman.kdex.dev/`. The suffix is arbitrary and used by developers to uniquely identify the gate on the workload (that gives 46 remaining characters to work with).

For example, if we have a dependency on a database, we could define it with the following _scheduling gate and annotation pair_:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
  annotations:
    gateman.kdex.dev/database: ...
spec:
  containers:
    - name: application
      image: nginx
  schedulingGates:
    - name: gateman.kdex.dev/database
```

The schema I've defined for the content of the annotation is as follows (stripped down to the essentials) (see [here](https://github.com/kdex-tech/kdex-gateman/blob/main/condition.schema.json) for the full schema):

```json
{
  "type": "object",
  "properties": {
    "apiVersion": {
      "type": "string"
    },
    "kind": {
      "type": "string"
    },
    "name": {
      "type": "string"
    },
    "namespace": {
      "type": "string"
    },
    "expression": {
      "type": "string"
    }
  },
  "required": [
    "apiVersion",
    "kind",
    "name"
  ]
}
```

This is called **the condition** and the `apiVersion`, `kind`, and `name` properties are used to identify the resource that the condition is about. The `namespace` property is used to specify the namespace of the resource. When omitted, the pod's namespace is used. The `expression` property is used to specify a [CEL](https://kubernetes.io/docs/reference/using-api/cel/) expression that must evaluate to true for the condition to be satisfied. The CEL environment contains two variables: `resource` and `pod`. `resource` gives access to the resource matching the `apiVersion`, `kind`, and `name` properties and `pod` gives access to the pod that is being scheduled. CEL expressions allow for complex conditions to be specified over the entire content of the pod and resource. When `expression` is omitted, the _existence_ of the resource is used to satisfy the condition. 

#### Example

Let's see what our original example looks like with **Gateman**. You'll recall that we had a database workload and a secret that the database workload depended on.

```yaml
# skip volumes
apiVersion: v1
kind: Pod
metadata:
  name: database
  annotations:
    gateman.kdex.dev/db-secret: |
      {
        "apiVersion": "v1",
        "kind": "Secret",
        "name": "db-secret"
      }
spec:
  containers:
    - name: database
      image: postgres:16
  schedulingGates:
    - name: gateman.kdex.dev/db-secret
```

Now the database pod has a scheduling gate for the secret and it won't be scheduled until the secret exists, perhaps indefinitely and without any resource waste.

This is a fairly simple example however. What if we have a more complex example?

Suppose that we combine the two earlier scenarios such that the database depends on the secret and the application depends both on the secret and the database.

```yaml
# skip volumes
apiVersion: v1
kind: Pod
metadata:
  name: database
  annotations:
    gateman.kdex.dev/db-secret: |
      {
        "apiVersion": "v1",
        "kind": "Secret",
        "name": "db-secret"
      }
spec:
  containers:
    - name: database
      image: postgres:16
  schedulingGates:
    - name: gateman.kdex.dev/db-secret
---
# skip volumes
apiVersion: v1
kind: Pod
metadata:
  name: application
  annotations:
    gateman.kdex.dev/db-pod: |
      {
        "apiVersion": "v1",
        "kind": "Pod",
        "name": "database",
        "expression": "resource.status.phase == 'Running'"
      }
    gateman.kdex.dev/db-secret: |
      {
        "apiVersion": "v1",
        "kind": "Secret",
        "name": "db-secret"
      }
spec:
  containers:
    - name: application
      image: nginx
  schedulingGates:
    - name: gateman.kdex.dev/db-pod
    - name: gateman.kdex.dev/db-secret
```

### Closing thoughts

Gateman is a controller that enables you to define scheduling gates and the conditions under which they should be removed in a declarative way using powerful CEL expressions. In doing so it allows you to define scalable, indefinite, resource free and failure free dependencies.

I'm excited to see where this goes and I'm looking forward to seeing what other people think about this idea. To that end, I've enabled [GitHub Discussions](https://github.com/kdex-tech/kdex-gateman/discussions) for the project where you can give feedback and share your thoughts and ideas. If you're interested in contributing to the project, please feel free to reach out to me there.