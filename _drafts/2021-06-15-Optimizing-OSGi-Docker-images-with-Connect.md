---
title: OSGi On Kubernetes: Optimizing Java Docker Images with OSGi
excerpt: Streamline your Java Docker images by leveraging the powers of OSGi build tools.
---

Today's Cloud environments enable organizations to quickly deploy applications without incurring up-front hardware costs by taking advantage of ready-to-use, pay-per-use infrastructure. However, pay-per-use billing, speed of deployment, and speed of startup each depend on the magnitude of the application components being deployed. It is no longer sufficient to ignore the size of images or to not optimize for start time. Furthermore, Java applications have long suffered the stigma of resulting in heavy, slow to start deployment components. While there are existing technologies in the Java ecosystem that tackle this aspect, they are far from invasive to your code. No longer. By taking advantage of OSGi and plain Java tooling we can product small and fast images that scale with the best of them.