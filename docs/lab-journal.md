Phase 1 - Clustering setup

* Simulated a production like environment by creating a multi-node kind cluster 1 control plane and 2 workers using the kind-config.yaml. This was done via
  2 separate nodes one for scheduling and one for fault tolerance.
* Created 3 namespaces to isolate workloads by function to follow best practices for RBAC: monitoring, security-tools, app-workloads.



Break #1 Bad Image Tag -

* Simulated a bad image by setting nginx:fakeimage which simulates a misconfigured deployment or typo in cicd pipeline.
* Pod Status: ErrImagePull -> ImagePullBackOff - Kubernetes retries downloading image but waits inbetween tries to prevent overloading image registry, this
  was caused by the wrong image tag in this case.
* Used kubectl describe to see "manifest unknown" which meant that Kubernetes could not find the image and was checking events section to see why the
  failure occurred.
* Fixed the set image nginx:1.25, it was good to understand how to fix this manually but in production this would be caught by cicd controllers.



Low Resource Limits

* Set memory limit to 1Mi which lead to new pods looping into failure.
* This showed me how old healthy pods won't be killed until the new pods are ready this causes the crash to go unnoticed at first as it was hidden.
* To see the broken pod I had to `kubectl scale --replicas=0' to force Kubernetes to delete everything. So that when it came back up the only pod was the
  new one with the low resources, which made it so that we can finally see the crash.
* Pod crashed with OOMKilled → CrashLoopBackOff
* I fixed this by seting limits to the memory to 128Mi memory and 250m CPU.
  kubectl set resources deployment/nginx -n app-workloads --limits=memory=128Mi,cpu=250m --requests=memory=64Mi,cpu=100m (the limit was put to prevent
  runaway containers)


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

* Default passwords did not work on fresh install and Helm ended up generating a random password instead of the default 'prom-operator' which was more
  secure and caught me off guard and led me down a debugging spree.
* To get the real password, I pulled it from the Kubernetes Secret with kubectl and decoded it in PowerShell (since the Linux base64 command doesn’t exist
  there). A good reminder that Kubernetes Secrets are only base64-encoded, not encrypted which leads for the need for tools like Sealed Secrets.  
  kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | ForEach-Object
  { \[System.Text.Encoding]::UTF8.GetString(\[System.Convert]::FromBase64String($\_)) }
* Also learned that kubectl port-forward svc/prometheus-grafana 3000:80 didn’t work, but port-forwarding directly to 3000:3000 did. This mattered for
  troubleshooting, since service routing adds an extra layer of abstraction that can fail independently from the pod itself.


Loki Data Source Connection

* Adding Loki was difficult it was showing "unable to connect" even though loki was healthy. This showed me UI errors do not always reflect the reality so
  verifying from inside the cluster is more reliable. kubectl get pods -n observability - Showed all Loki pods as Running
* Verified Loki was reachable Via kubectl exec -it grafana-pod -- curl http://loki:3100/ready - Returned Ready. This tested from inside the pod eliminates
  network policy and DNS issues as possible causes.
* Checked the Grafana API directly and found Loki was actually already added successfully further proving the UI error was misleading which is why checking
  APIs directly is a good debugging habit. 
* Queried nginx logs through Grafana Explore using the Loki data source and confirmed the full pipeline was working end-to-end verification is the only way 
  to confirm if a logging pipeline is actually functional.


Verification

* Grafana dashboards showed real time CPU and Memory metrics for the nginx pod with memory at 12.9MiB out of 128MiB limit (10%). This connects back to
  Phase 1 where 1Mi crashed the container and now I can see the resource usage in real time to prevent that. Nginx Metrics Dashboard in /images/grafana
  nginx-metrics.png
* Loki displayed nginx container logs showing the full startup sequence. This is what is important for incident response at scale being able to search logs 
  across all containers from one place.
* Full Pipeline: Container -> Fluent Bit -> Loki -> Grafana this is the same log architecture used in production environments I have worked in. 


Phase 3 - Kubernetes Hardening 


RBAC (Role Based Access Control)

* Created 3 roles to enforce least privilege access across the cluster: Analyst role (read only), Deployer(can update but not delete), and Admin(full 
  cluster access).
* Analyst role was limited to the monitoring namespace and could only view pods, logs, services and config maps. This follows the principle of least
  privilege used in real production environments, where visibility is necessary but changes are restricted.
* Deployer role was limited to the app-workloads namespace and allowed to create, update and patch resources, but not delete them. This helps prevent 
  accidental or malicious removal of production workloads while still letting ci/cd pipelines deploy changes.
* Admin role was created as a ClusterRole with full access across all namespaces. In a real production environment, this level of access would be tightly
  restricted and closely audited, but in this lab it was used to confirm that cross-namespace permissions were working correctly.
* Applied all roles using  kubectl apply -f security/rbac/ which applied all 3 role files at once. Keeping RBAC policies as YAML in Git means they are
  version controlled, auditable, and reproducible which aligns with NIST 800-53 AC-6(Least Privilege)

RBAC Tests

* Tested all 3 roles using the --as flag to impersonate each service account. These denied messages are proof that least-privilege is enforced.

* Analyst - read pods (worked):
  kubectl get pods -n monitoring --as=system:serviceaccount:monitoring:analyst-sa

* Analyst - delete pod (Forbidden):
  kubectl delete pod loki-0 -n monitoring --as=system:serviceaccount:monitoring:analyst-sa

* Deployer - list deployments (worked):
  kubectl get deployments -n app-workloads --as=system:serviceaccount:app-workloads:deployer-sa

* Deployer - delete deployment (Forbidden):
  kubectl delete deployment nginx -n app-workloads --as=system:serviceaccount:app-workloads:deployer-sa

* Admin - view all namespaces (worked):
  kubectl get pods --all-namespaces --as=system:serviceaccount:monitoring:admin-sa


Network Policies with Calico

* Default kind cluster uses kindnet as its CNI (container network interface) which does not support network policies. So I had to recreate the entire
  cluster with disableDefaultCNI: true in the kind config and install Calico to get real enforcement. This was worth it because having YAML files without
  the actual enforcement doesn't prove anything.
* Applied a default-deny-all policy on both monitoring and app-workloads namespaces blocking all ingress and egress traffic. This is the K8s equivalent of
  a firewall defaulting to deny-all which is the baseline for any secure environment.
* Created an allow-fluent-bit-to-loki policy that specifically allows Fluent Bit pods to send logs to Loki on port 3100. This is micro-segmentation, only
  the exact traffic that needs to flow is permitted.  
* Without that "directions" piece, the tools wouldn't be able to find anything else in the cluster, and the whole monitoring system would crash.

* Set up a security rule called "allow-monitoring." This does two main things: 
  Internal Talk: It lets all the monitoring tools talk to one another so they can share data.
  Finding the Way: It gives them permission to ask the system for directions (DNS).
  

Network Policy Tests

* Tested networking policies by exec'ing into the Grafana pod to verify traffic enforcement with Calico. These results prove cross-namespace traffic is
  blocked and intra-namespace traffic is allowed.

* Monitoring to Loki Intra-namespace worked and returned "ready" 
  kubectl exec -it deployment/prometheus-grafana -n monitoring -c grafana -- wget -qO- --timeout=5 http://loki.monitoring:3100/ready

* Monitoring to Nginx Cross-namespace (timed out, blocked):
  kubectl exec -it deployment/prometheus-grafana -n monitoring -c grafana -- wget -qO- --timeout=5 http://nginx.app-workloads:80

* Busybox in app-workloads - DNS resolution (failed, blocked):
  kubectl run test-pod --image=busybox --rm -it --restart=Never -n app-workloads -- wget -qO- --timeout=5 http://nginx.app-workloads


Pods Security Standards

* Labeled the app-workloads namespace with pod-security.kubernetes.io/enforce=restricted which tells Kubernetes to reject any pod that violates the
  restricted security profile. This is the strictest level and requires containers to drop all capabilities and run as non-root and set a seccomp(secure
  computing mode) profile

* When applying the label Kubernetes immediately warned that the existing nginx pod violates the restricted policy because it runs as root wit
  unrestricted capabilities. This is the same issue I discovered in Phase 1 break #3 where whoami returned root inside the container.

* Created a privileged-test.yaml with securityContext.privileged=true and then tried to deploy it. Kubernetes rejected it with a detailed error listing
  every violation 


Resource Quotas 

* Applied resource quotas to both monitoring and app-workloads namespaces to prevent any single namespace from consuming all cluster resources. Monitoring
  got 8Gi memory limit with 20 pod max, app-workloads got 4Gi memory limit with 10 pod max.
* Verified quotas with kubectl describe resourcequota which showed real usage vs limits. Monitoring had 11 pods using 400Mi of its 8Gi budget, app
  workloads had 1 pod using 64Mi of its 4Gi budget. This back to Phase 1 where resource misconfiguration crashed containers now there are guard rails at
  the namespace.
* Resource quotas map to NIST 800-53 SC-6 (Resource Availability) because they prevent one workload from starving others which is both a reliability and
  security concern.


Audit Logging

* Configured the Kubernetes API server to capture audit logs by creating an audit policy and modifying the kube-apiserver static pod manifest. This was the
  hardest part of phase 3 because it required editing the files inside the control plane container and any mistake crashes the API server.

* Created a tiered Audit Policy to balance security visibility with storage efficiency. 

  RequestResponse (High): Captured full details for 'Secrets' and 'Pod Exec' commands to ensure total traceability of sensitive data and user actions.
  Request (Medium): Logged all RBAC changes to track modifications to permissions and access control.
  Metadata (Low): Monitored Pods, Services, and ConfigMaps at the metadata level to track lifecycle changes without bloating logs.
  None: Excluded high-volume, 'noisy' endpoints and system events to reduce noise and optimize performance.

* Had to troubleshoot the API server crashing twice. 
  
  First time sed commands broke the YAML formatting by putting volume definitions in the wrong section.
  Second time the audit policy file was not accessible inside the API server container because it was not in a mounted path. 

* Fixed by copying the policy into /etc/kubernetes/pki/ which was already mounted.
* Verified audit logs are capturing events by checking /var/log/kubernetes/audit.log inside the control plane. Confirmed that the logs accurately capture
  the 'who, what, and when' of every system request, including authorization results and precise timestamps.This verification ensures alignment with NIST
  800-53 AU-2 (Audit Events) standards. Utilizied this command docker exec securelab-control-plane sh -c "tail -5 /var/log/kubernetes/audit.log"







