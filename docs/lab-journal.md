Phase 1 - Clustering setup

* Simulated a production like environment by creating a multi-node kind cluster 1 control plane and 2 workers using the kind-config.yaml. This was done via 2 separate nodes one for scheduling and one for fault tolerance.
* Created 3 namespaces to isolate workloads by function to follow best practices for RBAC: monitoring, security-tools, app-workloads.


Break #1 Bad Image Tag -

* Simulated a bad image by setting nginx:fakeimage which simulates a misconfigured deployment or typo in cicd pipeline.
* Pod Status: ErrImagePull -> ImagePullBackOff - Kubernetes retries downloading image but waits inbetween tries to prevent overloading image registry, this was caused by the wrong image tag in this case.
* Used kubectl describe to see "manifest unknown" which meant that Kubernetes could not find the image and was checking events section to see why the failure occurred.
* Fixed the set image nginx:1.25, it was good to understand how to fix this manually but in production this would be caught by cicd controllers.


Low Resource Limits

* Set memory limit to 1Mi which lead to new pods looping into failure.
* This showed me how old healthy pods won't be killed until the new pods are ready this causes the crash to go unnoticed at first as it was hidden. 
* To see the broken pod I had to `kubectl scale --replicas=0' to force Kubernetes to delete everything. So that when it came back up the only pod was the new one with the low resources, which made it so that we can finally see the crash. 
* Pod crashed with OOMKilled → CrashLoopBackOff
* I fixed this by seting limits to the memory to 128Mi memory and 250m CPU.
  kubectl set resources deployment/nginx -n app-workloads --limits=memory=128Mi,cpu=250m --requests=memory=64Mi,cpu=100m (the limit was put to prevent runaway containers)



Phase 2 - SIEM/Monitoring Stack Deployment


Prometheus Stack (Metrics Pipeline)
* Deployed Prometheus using Helm charts to scrape and store metrics like CPU, memory, and network from all pods and nodes. This acted as the central place to check metrics in realtime.
* 

