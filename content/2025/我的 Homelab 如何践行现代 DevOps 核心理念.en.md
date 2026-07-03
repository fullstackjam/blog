+++
title = "How My Homelab Embodies Modern DevOps Principles"
date = 2025-09-25
description = "Three years of homelab evolution: from snowflake servers to GitOps, from manual deployments to automated observability. A hands-on journey through Infrastructure as Code, ArgoCD, and Prometheus."
tags = ["homelab", "devops", "kubernetes", "gitops", "automation"]

[extra.comments]
issue_id = 12
[[extra.faq]]
question = "What hardware do I need to start a homelab?"
answer = "You don't need expensive gear. A few old laptops, mini PCs like Intel NUCs, or second-hand servers will do. My cluster started with a handful of cheap machines. The key is to get something running first — you can upgrade hardware later."

[[extra.faq]]
question = "Is Kubernetes overkill for a homelab?"
answer = "If you just want to run a few containers, Docker Compose is enough. But if you want to systematically learn cloud-native technologies, K8s is unavoidable. Once you have more than a handful of services, K8s declarative management and automatic scheduling actually become easier than managing everything manually."

[[extra.faq]]
question = "What is the difference between GitOps and traditional CI/CD?"
answer = "Traditional CI/CD uses a push model: the CI pipeline builds and pushes directly to the server. GitOps uses a pull model: ArgoCD inside the cluster continuously watches the Git repo and automatically syncs any changes. Git becomes the single source of truth, every change is tracked, and rollback is just a git revert."

[[extra.faq]]
question = "Does running a homelab help with career development?"
answer = "Absolutely. In interviews, you can talk about real problems you have solved instead of reciting textbook definitions. At work, issues like network policies, persistent volumes, and resource limits are things you have already debugged at home. Most importantly, you develop a DevOps mindset — that is worth more than knowing any single tool."

[[extra.faq]]
question = "What tech stack do you recommend for a homelab from scratch?"
answer = "Build it in stages. First, use Ansible for machine configuration to achieve Infrastructure as Code. Second, add K8s plus ArgoCD for GitOps automated deployment. Third, layer on Prometheus plus Grafana for observability. Get each stage working before moving to the next."
+++

Three years ago, when I started tinkering with [k8s-gitops](https://github.com/fullstackjam/k8s-gitops), I just wanted a Kubernetes cluster I could actually touch. Something to experiment with — break things, fix things, understand how they really work.

I didn't plan to end up with a production-grade setup that follows enterprise DevOps practices. That just sort of happened, one painful lesson at a time.

<!--more-->

---

## Why bother with a homelab?

Honestly? Curiosity.

At work, the K8s environment was locked down. I could look but not touch. Wanted to try a new feature? Get in the queue. Cloud services were convenient but expensive, and they abstract away the parts I wanted to understand.

The thing is, **reading docs only gets you so far. You have to break things yourself.**

Early on, I couldn't even explain the difference between a Pod and a Service. I read every tutorial I could find, but it all felt abstract. Then I started building from scratch — writing every config file, debugging every networking issue — and the concepts clicked. There's no shortcut for that.

---

## The three-year evolution

My homelab evolved across three phases:

{% mermaid() %}
graph LR
    subgraph "Phase 1: Manual Era"
        A[Manual SSH Setup] --> B[Snowflake Servers]
        B --> C[Manual Deployments]
    end

    subgraph "Phase 2: Automation"
        D[Ansible Playbooks] --> E[Standardized Nodes]
        E --> F[K8s Cluster]
    end

    subgraph "Phase 3: GitOps"
        G[Git Repository] --> H[ArgoCD Sync]
        H --> I[Auto Deploy]
        I --> J[Prometheus Metrics]
        J --> K[Grafana Dashboards]
    end

    C -.->|"Had enough"| D
    F -.->|"Manual deploys are painful"| G

    style A fill:#FEE2E2,stroke:#EF4444
    style B fill:#FEE2E2,stroke:#EF4444
    style C fill:#FEE2E2,stroke:#EF4444
    style D fill:#FEF3C7,stroke:#F59E0B
    style E fill:#FEF3C7,stroke:#F59E0B
    style F fill:#FEF3C7,stroke:#F59E0B
    style G fill:#D1FAE5,stroke:#10B981
    style H fill:#D1FAE5,stroke:#10B981
    style I fill:#D1FAE5,stroke:#10B981
    style J fill:#DBEAFE,stroke:#3B82F6
    style K fill:#DBEAFE,stroke:#3B82F6
{% end %}

Each phase was born out of frustration with the previous one. Let me walk through them.

---

## Phase 1: Killing the snowflake servers

### The problem: every machine was a unique work of art

In the beginning, every machine was hand-configured. One had Docker 20.10, another had Docker 24.0. One had iptables rules I added during a late-night debugging session. Another had mystery entries in `/etc/hosts` that nobody could explain. Config files were scattered everywhere, versions were inconsistent.

This is the classic "snowflake server" problem — every machine is unique, nobody dares touch them, and when something breaks, nobody can reproduce the environment.

When a machine died, the only option was a fresh OS install. And after reinstalling, I could never remember exactly what I had configured before.

### The fix: Infrastructure as Code with Ansible

I switched to **Ansible**. Everything from OS setup to software configuration became a playbook.

```yaml
# Node initialization playbook (simplified)
- hosts: k8s_nodes
  tasks:
    - name: Update system packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install base dependencies
      apt:
        name:
          - curl
          - apt-transport-https
          - ca-certificates
        state: present

    - name: Configure kernel parameters
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
      loop:
        - { key: net.bridge.bridge-nf-call-iptables, value: "1" }
        - { key: net.ipv4.ip_forward, value: "1" }

    - name: Install containerd
      apt:
        name: containerd
        state: present
```

Now rebuilding the entire cluster is one command:

```bash
ansible-playbook -i inventory/hosts site.yml
```

Five minutes and the cluster is back online. More importantly, every machine has the exact same configuration. No more "this server is special" excuses.

**Biggest takeaway:** Once your infrastructure is code, you stop being afraid of breaking things. You can always rebuild. The config lives in Git, so who changed what and when is always clear.

---

## Phase 2: GitOps saved my sanity

### The problem: manual deployments are a nightmare

The K8s cluster was up. Apps were running. But every update meant SSH-ing into the server and running commands by hand:

```bash
# The "standard" deployment process
ssh master-node
cd /home/user/manifests
git pull
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app
# Hope nothing goes wrong...
```

I would forget steps. Services would go down. Worse, I had no idea which version was actually running in production. One time I spent two hours debugging an issue, only to realize the "deployed" code was three days old — I had forgotten to actually deploy it.

### The fix: ArgoCD + declarative config

**ArgoCD** changed everything.

The concept is simple: ArgoCD runs inside the cluster and continuously watches a Git repository. You describe the desired state as YAML, push to Git, and ArgoCD detects the change and syncs the cluster to match.

```yaml
# ArgoCD Application definition
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homepage
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/fullstackjam/k8s-gitops
    targetRevision: main
    path: apps/homepage
  destination:
    server: https://kubernetes.default.svc
    namespace: homepage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Now deployment is just a git push. Rollback? `git revert` and push — ArgoCD automatically restores the previous state. Every deployment has a full Git history, so "what changed yesterday?" is always answerable.

I set up different sync strategies for different kinds of workloads:

- **Infrastructure components** (cert-manager, ingress-nginx): Manual sync. Validate in staging first.
- **My own apps** (blog, APIs): Auto sync. Push and ship.
- **Third-party services** (Grafana, Prometheus): Auto sync + selfHeal, preventing config drift from manual changes.

**Deepest lesson:** When deployment becomes a Git operation, the operational burden drops by half. No commands to memorize, no fear of fat-fingering something, and `git log` becomes your incident timeline.

---

## Phase 3: Observability so I can sleep at night

### The problem: the system was a black box

Before monitoring, troubleshooting was guesswork. CPU high? No idea. Running out of memory? Also no idea. I only found out services were down when someone told me.

My blog was down for two full days once and I had no clue — I don't check it every day. A friend finally messaged me: "Hey, your blog is broken."

### The fix: Prometheus + Grafana + alerting

A proper monitoring stack made the entire system transparent.

**Prometheus** collects metrics — CPU, memory, and disk for every node; resource usage for every Pod; request latency and error rates for every Service.

**Grafana** visualizes it all. I built dashboards for different concerns:

- **Cluster overview:** Node status, resource utilization, Pod counts
- **Application monitoring:** CPU/memory trends per service, restart counts
- **Network monitoring:** Ingress traffic, latency distribution

The most important piece is **alerting**. I configured AlertManager to notify me when things go sideways:

```yaml
# Alert rules (simplified)
groups:
  - name: node-alerts
    rules:
      - alert: NodeHighCPU
        expr: node_cpu_seconds_total > 0.85
        for: 5m
        annotations:
          summary: "Node CPU usage above 85%"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        annotations:
          summary: "Pod is crash-looping"

      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.15
        for: 10m
        annotations:
          summary: "Disk space below 15%"
```

Now I get warnings before problems become outages. Last time a node's disk was filling up, I got an alert a full day before it would have caused issues. Cleared some logs, moved on with my life. Before monitoring, that node would have gone down and taken every Pod on it with it.

**Most practical lesson:** Good monitoring isn't about showing off dashboards. It's about sleeping through the night.

---

## The real takeaway: DevOps is a mindset, not a toolbox

After three years, the biggest value of this project isn't the technology. It's the shift in thinking.

I used to think DevOps was just a collection of tools — learn Docker, K8s, and CI/CD, and you're done. Actually doing it taught me that DevOps is a way of thinking:

- **Automate everything that can be automated** — Ansible for config, ArgoCD for deployments, Renovate Bot for dependency updates
- **Monitor everything that runs** — A service without metrics is running blind
- **Version-control everything** — All configuration lives in Git. What changed, why, and who did it are always answerable
- **Make everything reproducible** — Anyone with the code can rebuild the entire environment

Once these principles are internalized, you see every technical problem differently. Now whenever I build anything, the first questions are: "Can this be automated?" "Is there monitoring?" "Is this configuration versioned?"

---

## Career impact

On a more practical note, this project has been a career accelerator.

In interviews, while others are reciting textbook definitions, I can talk naturally about real problems and solutions. Why network policies need explicit `policyTypes`. Why PV-to-PVC binding gets stuck. Why setting resource limits too low triggers OOMKill. These aren't abstract concepts anymore — they're bugs I have personally debugged.

At work, when similar issues come up, I can usually pinpoint the cause quickly. Because I have already seen it in my homelab.

---

## Advice for getting started

If you're thinking about building a homelab, here's what I've learned:

**Don't overcomplicate it from the start.** I made this mistake — I tried to set up K8s, Istio, Vault, and ArgoCD all at once and ended up with a mess. Build incrementally. Suggested order: Docker, then K8s, then GitOps, then monitoring, then everything else.

**Pick one project and go deep.** Don't bounce between Docker today, K8s tomorrow, and Terraform the day after. I stuck with one goal — "run a complete K8s cluster at home" — and followed it all the way through.

**Document every problem you solve.** I still regularly reference my old notes. Many solutions are in there. And those notes become your best interview material.

**Don't be afraid to break things.** It's your own environment. I've reinstalled the OS at least twenty times over three years, and every time I learned something new. Once Ansible was in place, reinstalling stopped being painful — one command and I'm back.

**Don't wait for the perfect architecture.** I've seen too many people stuck in planning mode. Start with a single-node K8s cluster. Get something running. Fix problems as they come up.

**Join the community.** The Kubernetes community is full of experienced people. Search "k8s-gitops" or "homelab" on GitHub for excellent reference projects.

---

## Final thoughts

[k8s-gitops](https://github.com/fullstackjam/k8s-gitops) proves one thing: personal projects can have production-grade quality. The difference isn't scale — it's principles and practice.

Through Infrastructure as Code, GitOps, and observability, you can build systems that are stable, reliable, and maintainable. And everything you learn along the way transfers directly to your day job.

If you're considering a homelab, don't hesitate. Grab an old computer and start. Three years from now, you'll be glad you did.

---

**Project:** [https://github.com/fullstackjam/k8s-gitops](https://github.com/fullstackjam/k8s-gitops)

**Docs:** [https://k8s-gitops.fullstackjam.com](https://k8s-gitops.fullstackjam.com)

Questions and discussions welcome on GitHub. Stars appreciated.
