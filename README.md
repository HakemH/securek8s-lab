# securek8s-lab
Kubernetes security lab — RBAC, network policies, admission control, container image scanning, and runtime threat detection for a containerized SIEM stack.

flowchart TB
    %% ===== Cluster Boundary =====
    subgraph CLUSTER["Kind Cluster (securelab)"]
        direction TB

        %% ===== Monitoring Namespace =====
        subgraph MON["Monitoring Namespace"]
            direction LR
            Prom[Prometheus]
            Graf[Grafana]
            Loki[Loki]
            FB[Fluent Bit]

            Prom --> Graf
            Loki --> Graf
            FB --> Loki

            AM[Alertmanager]
            KSM[Kube-State-Metrics]
            NE[Node Exporter]

            Prom --> AM
            Prom --> KSM
            Prom --> NE
        end

        %% ===== Network Policy Barrier =====
        NP["🚫 Default-Deny Network Policy"]

        %% ===== App Workloads Namespace =====
        subgraph APP["App-Workloads Namespace"]
            direction TB
            NGINX["NGINX App\n(resource-limited)"]
            PSP["Pod Security\nrestricted"]
            PSP --> NGINX
        end

        %% ===== Security Controls =====
        subgraph SEC["Security Controls"]
            direction TB
            RBAC["RBAC\n(analyst / deployer / admin)"]
            CALICO["Calico CNI\nNetwork Policies"]
            PSS["Pod Security Standards\nrestricted"]
            QUOTA["Resource Quotas\nper-namespace"]
            AUDIT["Audit Logging\nAPI Server"]
        end

        %% ===== Node Info =====
        NODES["Nodes\n1 control-plane\n2 workers"]

        %% ===== Relationships =====
        MON --> NP --> APP
        CALICO --> NP
        SEC --> APP
    end

    %% ===== Styling =====
    classDef namespace fill:#111827,stroke:#6366f1,stroke-width:2px,color:#e5e7eb;
    classDef service fill:#1f2937,stroke:#22d3ee,stroke-width:1.5px,color:#e5e7eb;
    classDef security fill:#18181b,stroke:#f59e0b,stroke-width:2px,color:#fde68a;
    classDef warning fill:#450a0a,stroke:#ef4444,stroke-width:2px,color:#fecaca;

    class MON,APP namespace
    class Prom,Graf,Loki,FB,AM,KSM,NE,NGINX service
    class RBAC,CALICO,PSS,QUOTA,AUDIT security
    class NP warning
