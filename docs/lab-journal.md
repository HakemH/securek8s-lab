The first phase I learned




Clustering setup

* Simulated a production like environment by creating a multi-node kind cluster 1 control plane and 2 workers using the kind-config.yaml. This was done via 2 separate nodes one for scheduling and one for fault tolerance.
* Created 3 namespaces to isolate workloads by function to follow best practices for RBAC: monitoring, security-tools, app-workloads.

  



Break #1 Bad Image Tag - 



* Simulated a bad image by setting nginx:fakeimage which simulates a misconfigured deployment or typo in cicd pipeline.
* Pod Status: ErrImagePull -> ImagePullBackOff - Kubernetes retries downloading image but waits inbetween tries to prevent overloading image registry, this was caused by the wrong image tag in this case.
* Used kubectl describe to see "manifest unknown" which meant that Kubernetes could not find the image and was checking events section to see why the failure occurred.
* Fixed the set image nginx:1.25, it was good to understand how to fix this manually but in production this would be caught by cicd controllers.
