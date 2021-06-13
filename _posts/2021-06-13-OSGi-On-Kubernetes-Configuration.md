---
title: OSGi On Kubernetes - Configuration
excerpt: Supercharge your Kubernetes apps with real-time configuration updates using OSGi Configuration Admin.
---

Configuration is an important aspect in application design and especially in the design of Cloud applications. The need to develop, test, and release code in the ephemeral spaces of the Cloud mean that many important details about environment and behaviour need to be abstracted away from the application logic.

Many years before Cloud environments existed [OSGi](https://www.osgi.org/) technology was being leveraged to deploy complex, distributed applications to countless devices. Due in large part to its resilience and innovative design it continues to be used in this fashion today. Relatively early in OSGi's long history the [Configuration Admin Service Specification](https://docs.osgi.org/specification/osgi.cmpn/7.0.0/service.cm.html) was defined with much the same concerns for configuration as are present in current environments like today's Cloud.

[Kubernetes](https://kubernetes.io) is a very popular and widely adopted technology for orchestrating applications in the Cloud and is provided by a growing number of the largest Cloud vendors. It also defines a set of core components that are responsible for keeping applications well fed with configuration.

Notably, Kubernetes configuration principles very closely resemble those of OSGi Configuration making integration quite natural which is the topic of this post.

## Defining ConfigMaps

The first Kubernetes configuration component is called [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) and represents the basic building block for providing non-confidential data in key-value pairs.

_(Don't place secure or data that requires encryption in ConfigMaps. Another Kubernetes component we're going to talk about later is intended for that.)_

A ConfigMap is defined in a Kubernetes manifest file (commonly using the `YAML` syntax) like the following:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osgi-demo-config
data:
  # property-like keys; each key maps to a simple value
  pool_initial_size: "3"
  pool_max_size: "10"

  # file-like keys
  message.processor-email.config: |
    pool.initial.size=i"${env:pool_initial_size}"
	pool.max.size=i"${env:pool_max_size}"
    topic="/email"
  message.processor-log.config: |
    pool.initial.size=i"${env:pool_initial_size}"
	pool.max.size=i"20"
    topic="/log"
```

_(We'll be seeing a lot of `YAML` in this and future posts so take heart.)_

The example above demonstrates the basic model for ConfigMaps. The first 4 lines are boiler plate and are setup for the `data` field which holds a series of key-value pairs in one of two flavours;

- **property-like keys** - typical key mapping to a single value
- **file like keys** - maps a key (typically with a file-like name) to a multi-line chunk of data (that looks a lot like a file)

***Take note*** *of the syntax used in the file content. We'll come back to that later.*

The next thing you should note about the value of the *file-like-keys* flavour is that the format of the value is largely irrelevant. The ConfigMap does not care one way or the other if it's XML, JSON, Java properties files, more YAML, or any other character based syntax you wish (I supposed it could even be script... but I'd be careful with that). The only real restriction is that a given ConfigMap's `data` does not exceed 1 [MiB](https://simple.wikipedia.org/wiki/Mebibyte) in size.

## Consuming ConfigMaps

There are several ways this could be used but we'll talk about the 2 most common.

Consider the following [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) spec.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: osgi-demo-pod
spec:
  containers:
    - name: osgi-demo-container
      image: rotty3000/config-osgi-k8s-demo
      env:
        # Define environment variables
        - name: pool_initial_size
          valueFrom:
            configMapKeyRef:
              name: osgi-demo-config # The ConfigMap this value comes from
              key: pool_initial_size # The key to fetch
        - name: pool_max_size
          valueFrom:
            configMapKeyRef:
              name: osgi-demo-config
              key: pool_max_size
      volumeMounts:
      - name: config
        mountPath: "/app/configs"
        readOnly: true
  volumes:
    - name: config
      configMap:
        # The name of the ConfigMap you want to mount
        name: osgi-demo-config
```

_(Pods are the smallest deployable units of computing in a Kubernetes cluster.)_

The example above defines a Pod, the first 5 lines of which are pretty boiler plate, named `osgi-demo-pod`, has exactly one `container` named `osgi-demo-container`. The container specifies the Docker image to be `rotty3000/config-osgi-k8s-demo`. That alone is the basic information necessary to define a usable Pod. You could deploy this and have a running system:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: osgi-demo-pod
spec:
  containers:
    - name: osgi-demo-container
      image: rotty3000/config-osgi-k8s-demo
```

_(Though [Naked Pods](https://kubernetes.io/docs/concepts/configuration/overview/#naked-pods-vs-replicasets-deployments-and-jobs) are discouraged, they are useful for illustration.)_

However, this is not sufficient in the majority of cases. Usually some additional input is required, such as resource constraints, volume mounts, configuration, etc.

Let's start our examination at the bottom of the spec with the `volumes` field. [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) are Kubernetes' mechanism for sharing file systems that supplement a Pod's own ephemeral file system (ephemeral meaning nothing that is modified in the Pod's own file system is retained across Pod restarts). There are many different types of Volumes in Kubernetes. The one we're showing here adds a ConfigMap:

```yaml
  volumes:
    - name: config
      configMap:
        # The name of the ConfigMap you want to mount
        name: osgi-demo-config
```

In order for the ConfigMap to be usable it must be mounted as a volume, so we define the volume by first giving it a name (`config`), by specifying a Volume _type_ in this case a `configMap` named (`osgi-demo-config`).

_(The Pod and the ConfigMap have to be in the same [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).)_

Now that the volume is defined we can *mount* it into the pod using the `volumeMounts` field under the `containers` array:

```yaml
      volumeMounts:
      - name: config
        mountPath: "/app/configs"
        readOnly: true
```

The mount is set to be located at `/app/configs`. In this case the image we're using already listens to this directory using [Apache Felix FileInstall](https://felix.apache.org/documentation/subprojects/apache-felix-file-install.html); an OSGi bundle that manages OSGi configurations from the file system on demand. A perfect companion for Kubernetes ConfigMaps. FileInstall is configured to do this using the system property `felix.fileinstall.dir=/app/configs`.

### Mounted files

You should note that as configured the volume will cause _each key_ in the ConfigMap to be mounted as a separate file in the mount directory (the value of each pair being the contents of the file). This is great because FileInstall automatically detects and manages files that end with `.config` (or the older `.cfg`) identified in it's scanned directory	. _(Unknown files will be ignored.)_

#### Mounting specific entries

You _can_ pick and choose which keys of the ConfigMap are loaded as files by adding an `items` array:

```yaml
  volumes:
    - name: config
      configMap:
        # The name of the ConfigMap you want to mount
        name: osgi-demo-config
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "message.processor-email.config"
          path: "message.processor-email.config"
        - key: "message.processor-log.config"
          path: "message.processor-log.config"
```

However; I don't recommend the `items` approach in the OSGi Configuration scenario in the default case because it forces a duplication of the same information in two places; the ConfigMap and the `volumes` definition. I suggest lettings the ConfigMap be the source of truth and I'll explain why a little later.

#### Sub-directory scans

By default FileInstall scans sub-directories as well, so create any hierarchy that suites your needs by mounting the ConfigMap in any sub-directory of the path scanned by FileInstall. This is handy for grouping related configuration files.

### Environment Variables

Another way to use the key-value pairs of the ConfigMap in the Pod is by defining environment variables. If you go back to the `env` field of the Pod spec you'll see the following:

```yaml
      env:
        # Define environment variables
        - name: pool_initial_size
          valueFrom:
            configMapKeyRef:
              name: osgi-demo-config # The ConfigMap this value comes from
              key: pool_initial_size # The key to use
        - name: pool_max_size
          valueFrom:
            configMapKeyRef:
              name: osgi-demo-config
              key: pool_max_size
```

Here, two of the keys are cherry-picked from the ConfigMap and turned into environment variables that will be available in the Pod from the moment of startup.

Interestingly, if you recall the note about the syntax of the configuration files content which I've replicated next,

```yaml
  message.processor-email.config: |
    pool.initial.size=i"${env:pool_initial_size}"
	pool.max.size=i"${env:pool_max_size}"
    topic="/email"
```

you should be aware that FileInstall has built in interpolation for environment variables using the `env:` prefix. This allows you to compose configuration using the environment with no additional tooling installed.

### Dynamic reaction to configuration changes

Many software systems are capable of gracefully handling changes in configuration without requiring full restarts. For example [Apache HTTPD Server](https://httpd.apache.org) can use it's [`graceful`](https://httpd.apache.org/docs/current/stopping.html#graceful) restart mode to smoothly reload it's configuration and automatically adjust its behaviour accordingly. [NGINX](https://docs.nginx.com/nginx/admin-guide/basic-functionality/runtime-control/) offers a similar functionality and so do many other systems.

Besides the obvious goal of reducing service interruptions this also supports the notion of feature flags, service discovery and other real-time signals required to efficiently and speedily control system behaviour.

### ConfigMaps as mounted files vs. environment variables

Given that [mounted ConfigMaps are updated automatically](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically) without needing to restart the Pod vs. environment variable which require a Pod be restarted in order to be updated means that mounted files are more efficient for systems that are designed to be signalled on configuration file changes.

## Conclusion

**OSGi Configuration Admin** is meant to dynamically propagate configuration changes throughout an OSGi framework. **Apache Felix FileInstall** adds to that the ability to react to changes in configuration resources on the file system which are subsequently propagated to Configuration Admin. **Kubernetes ConfigMaps** provide Pods with centrally and dynamically updated configuration resources.

The combination of the three makes for a completely dynamic and fully integrated application configuration system with very little effort.

In a future post I plan to further various aspects of this subject. So stay tuned!

### Test Resources

The project demonstrating the ideas in this post can be found in [this repository](https://github.com/rotty3000/osgi-config-aff). The docker image that defines the configurable OSGi application can be found [here](https://hub.docker.com/r/rotty3000/config-osgi-k8s-demo) on Docker Hub.



