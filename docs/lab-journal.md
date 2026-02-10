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
* Utilized Grafana as the dashboard to present both metrics and logs, helped having everything in one place.
* Alertmanager handles alerts from Prometheus rules and routes notifications. In a production environment this would page the on-call team so issues get caught quickly.
* Kube-state-metrics exposes Kubernetes object state like deployment health, pod phases and replica counts as Prometheus metrics. Without this Prometheus only checks for resource usage which would not show you the in-depth as to which pods/deployments are failing.
* Utilized this command "helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring" 
* Helm bundled Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics together, to simulate best practice as how production teams deploy monitoring stacks since it keeps everything version-compatible and reproducible.



Loki Stack (Logging Pipeline)

* Deployed Loki as a lightweight log aggregation system that indexes and stores logs — centralized logging is essential for security investigations because you need to search across all containers not just one at a time. In production I have used Logscale at Crowdstrike and Splunk at Oracle.
* Fluent bit runs as a Daemonset on each worker node to collect container stdout/stderr logs and ship them to Loki. Utilized a DaemonSet to ensure every node has a log collector so no logs are missed when new nodes are added.



Pipeline Logic

* Fluent Bit collects logs -> ships to Loki -> Grafana queries Loki for log exploration (which mirrors what SIEM Prod environments where agents collect) -> This stores in a backend and then frontend is used for queries.
* Node-exporter collects the hardware metrics -> Prometheus scrapes it -> Grafana shows cluster health, this gives visibility into Kubernetes specific events like failed deployments or evicted pods that raw resource wouldn't catch



OpenSearch Crash 

* Originally tried to deploy OpenSearch/ELK as the SIEM but it was too heavy for the environment. The issue that I faced was that there was network issues inside the kind cluster, this showed me that init containers can block the entire pod startup if they fail.
* The issue with OpenSearch was that it had a high memory requirements which caused Docker Desktop to crash which corrupted the kind cluster entirely. This is why resource planning matters before deploying to any cluster especially in constrained environments.
* Fixed this issue by having to `kind delete cluster --name securelab` and recreate from scratch using my saved kind-config.yaml. The saved config in Git meant that I can rebuild in minutes instead of having to start over from memory. 
* Had to Rebuild all of phase 1 (namespaces, nginx deployment) before redeploying the monitoring stack, this reinforced the importance of infrastructure as code since everything was reproducible from saved manifests.
* Ended up switching to Loki which is much lighter and integrates directly with Grafana. In production OpenSearch would have the right-size for the environment but for my local setup Loki was the better choice as it saved a lot of memory.


Grafana Password Issues

* Default passwords did not work on fresh install and Helm ended up generating a random password instead of the default 'prom-operator' which was more secure and caught me off guard and led me down a debugging spree.
* To get the real password, I pulled it from the Kubernetes Secret with kubectl and decoded it in PowerShell (since the Linux base64 command doesn’t exist there). A good reminder that Kubernetes Secrets are only base64-encoded, not encrypted which leads for the need for tools like Sealed Secrets.  kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | ForEach-Object { \[System.Text.Encoding]::UTF8.GetString(\[System.Convert]::FromBase64String($\_)) }
* Also learned that kubectl port-forward svc/prometheus-grafana 3000:80 didn’t work, but port-forwarding directly to 3000:3000 did. This mattered for troubleshooting, since service routing adds an extra layer of abstraction that can fail independently from the pod itself.



Loki Data Source Connection

* Adding Loki was difficult it was showing "unable to connect" even though loki was healthy. This showed me UI errors do not always reflect the reality so verifying from inside the cluster is more reliable. kubectl get pods -n observability - Showed all Loki pods as Running
* Verified Loki was reachable Via kubectl exec -it grafana-pod -- curl http://loki:3100/ready - Returned Ready. This tested from inside the pod eliminates network policy and DNS issues as possible causes.
* Checked the Grafana API directly and found Loki was actually already added successfully further proving the UI error was misleading which is why checking APIs directly is a good debugging habit. 
* Queried nginx logs through Grafana Explore using the Loki data source and confirmed the full pipeline was working end-to-end verification is the only way to confirm if a logging pipeline is actually functional.



Verification

* Grafana dashboards showed real time CPU and Memory metrics for the nginx pod with memory at 12.9MiB out of 128MiB limit (10%). This connects back to Phase 1 where 1Mi crashed the container and now I can see the resource usage in real time to prevent that. Nginx Metrics Dashboard in /images/grafana-nginx-metrics.png
* Loki displayed nginx container logs showing the full startup sequence. This is what is important for incident response at scale being able to search logs across all containers from one place.
* Full Pipeline: Container -> Fluent Bit -> Loki -> Grafana this is the same log architecture used in production environments I have worked in. 



