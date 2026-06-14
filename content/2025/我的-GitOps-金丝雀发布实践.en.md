+++
title = "My GitOps Canary Deployment Journey: Argo Rollouts with Gateway API"
date = 2025-11-27
description = "Building a canary deployment pipeline from scratch with Argo Rollouts and Gateway API -- automated traffic splitting, metric analysis, and rollback for a Homelab GitOps setup"
tags = ["gitops", "canary", "kubernetes", "argo-rollouts", "gateway-api", "traefik", "prometheus"]

[extra.comments]
issue_id = 13

[[extra.faq]]
question = "What is the difference between canary deployment and blue-green deployment?"
answer = "Blue-green maintains two full environments and switches all traffic at once. Canary is gradual -- you route a small percentage of traffic (say 5-20%) to the new version first, monitor metrics, and only increase traffic if things look healthy. Argo Rollouts supports both strategies."

[[extra.faq]]
question = "Why use Gateway API instead of Ingress for traffic splitting?"
answer = "Ingress relies on vendor-specific annotations for traffic splitting -- each controller has its own syntax. Gateway API is the official Kubernetes successor to Ingress, with native weight-based routing in HTTPRoute. It is standardized across controllers, so you can swap implementations without rewriting configs."

[[extra.faq]]
question = "How do Argo Rollouts and ArgoCD relate to each other?"
answer = "ArgoCD handles GitOps synchronization, ensuring your cluster state matches your Git repository. Argo Rollouts handles release strategy, controlling how new versions roll out. They are independent but complementary: ArgoCD manages 'what to deploy', Rollouts manages 'how to release'."

[[extra.faq]]
question = "How can I test canary deployments locally?"
answer = "You can use Kind or k3d to create a local Kubernetes cluster, then install Traefik, Argo Rollouts, and Prometheus. Allocate at least 8GB of memory to Docker since the full stack is resource-hungry. The project repository includes a complete local setup guide."

[[extra.faq]]
question = "What happens when a PromQL query returns no data during canary analysis?"
answer = "This is a common pitfall. When the canary has no traffic yet, PromQL returns empty results, which Argo Rollouts treats as a failure and triggers rollback. The fix is to add 'or vector(1)' to your query, which defaults to 100% success rate when there is no data."
+++

Remember my [k8s-gitops](https://github.com/fullstackjam/k8s-gitops) project? It brought Infrastructure as Code to my Homelab -- ArgoCD keeps the cluster in sync with the Git repo, and the question of "how do I deploy things" was finally solved.

But after running it for a few months, a new anxiety crept in: **how do I update things safely?**

<!--more-->

Every time I pushed a change, ArgoCD would sync it, and the new version would go live for everyone immediately. My mental state during each deployment could be summed up in two words: pure hope. What if there is a bug? What if a config value is wrong? Every user (mainly my family and friends, but still) gets hit instantly, and rolling back means scrambling through kubectl commands.

That got me thinking: what if I could do what the big tech companies do -- route a small slice of traffic to the new version first, verify it is healthy, and then gradually ramp up?

That is how this project was born: [canary-deployment](https://github.com/fullstackjam/canary-deployment).

---

## The problem with all-at-once releases

Why "big bang" deployments are nerve-wracking.

A standard Kubernetes Deployment rolling update is not instantaneous, but new Pods start receiving traffic the moment they pass readiness checks. The whole rollout takes maybe 30 seconds. If the new version has a problem, by the time you notice, a significant chunk of requests has already hit the broken Pods.

Worse, rolling back depends on you running `kubectl rollout undo` or manually syncing ArgoCD to the previous version. At 2 AM when an alert wakes you up, are you confident you will get it right on the first try?

What I wanted was a flow like this:

{% mermaid() %}
graph LR
    A[Git Push] --> B[ArgoCD Sync]
    B --> C[Create Canary Pod]
    C --> D[Route 20% Traffic]
    D --> E{Metric Analysis}
    E -->|Healthy| F[Route 50% Traffic]
    F --> G{Metric Analysis}
    G -->|Healthy| H[Full Rollout]
    E -->|Unhealthy| I[Auto Rollback]
    G -->|Unhealthy| I

    style A fill:#DBEAFE,stroke:#2563EB,color:#1E40AF
    style B fill:#DBEAFE,stroke:#2563EB,color:#1E40AF
    style C fill:#D1FAE5,stroke:#059669,color:#065F46
    style D fill:#D1FAE5,stroke:#059669,color:#065F46
    style F fill:#D1FAE5,stroke:#059669,color:#065F46
    style H fill:#D1FAE5,stroke:#059669,color:#065F46
    style E fill:#FEF3C7,stroke:#D97706,color:#92400E
    style G fill:#FEF3C7,stroke:#D97706,color:#92400E
    style I fill:#FEE2E2,stroke:#DC2626,color:#991B1B
{% end %}

Push code, and the system automatically handles the full loop of probe, observe, promote or abort. No human in the loop. That is what GitOps should look like.

---

## The tech stack

To build this flow, I needed three components, each with a clear job:

**Argo Rollouts** -- the progressive delivery controller. It extends Kubernetes with a `Rollout` resource that lets you define canary or blue-green release strategies. Think of it as the brain that decides "how much traffic to shift, when to advance, and when to roll back."

**Gateway API + Traefik** -- the traffic splitter. Gateway API is the official Kubernetes successor to Ingress, and its `HTTPRoute` resource natively supports a `weight` field for traffic distribution. Traefik serves as the Gateway Controller that does the actual routing. Unlike the Ingress era where every controller had its own proprietary annotations for traffic splitting, Gateway API is standardized.

**Prometheus** -- the metrics engine. It answers the critical question: "Is the new version actually working?" Argo Rollouts periodically queries Prometheus and uses the results to decide whether to continue or roll back.

The division of labor is clean: Rollouts makes decisions, Gateway API executes them, Prometheus provides the data.

---

## Implementation

### Rollout: redefining "deploy"

With Argo Rollouts, you swap out `Deployment` for `Rollout`. It looks almost identical, except for the crucial `strategy` field:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  strategy:
    canary:
      stableService: demo-app-stable
      canaryService: demo-app-canary
      trafficRouting:
        plugins:
          argoproj-labs/gatewayapi:
            httpRoute: demo-app-route
            namespace: default
      steps:
        - setWeight: 20
        - pause: {duration: 30s}
        - setWeight: 50
        - pause: {duration: 30s}
        - setWeight: 100
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: demo-app:v2
          ports:
            - containerPort: 8080
```

This defines a three-phase release strategy:

1. When the image is updated, create new Pods and shift 20% of traffic to them
2. Pause for 30 seconds while Prometheus collects metrics
3. If metrics look good, ramp up to 50% and observe for another 30 seconds
4. Finally, go to 100%

Notice the two Services: `demo-app-stable` points to the old Pods, `demo-app-canary` points to the new ones. Argo Rollouts controls traffic distribution by modifying the weight values in the HTTPRoute.

### AnalysisTemplate: data-driven decisions

Traffic splitting is only half the story. The real question is: **who decides whether to continue or roll back?**

A human staring at Grafana dashboards? At 3 AM?

`AnalysisTemplate` is Argo Rollouts' automated referee. Here is a success-rate check:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 3
      successCondition: result[0] >= 0.99
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus-server.prometheus-system.svc.cluster.local:80
          query: |
            sum(rate(http_requests_total{status!~"5.*"}[1m]))
            /
            sum(rate(http_requests_total[1m]))
```

This template tells Argo Rollouts:

- Query Prometheus every 1 minute
- Run 3 checks total
- If the HTTP success rate drops below 99%, mark it as a failure
- Allow at most 1 failure before triggering rollback

Then you wire it into the Rollout steps:

```yaml
steps:
  - setWeight: 20
  - analysis:
      templates:
        - templateName: success-rate
  - setWeight: 50
  - pause: {duration: 30s}
  - setWeight: 100
```

After traffic shifts to 20%, the analysis runs automatically. It only advances if the analysis passes. If it fails, traffic reverts to the stable version. No pager duty required.

### Gateway API: standardized traffic splitting

In the Ingress days, canary deployments with Nginx meant writing `nginx.ingress.kubernetes.io/canary-weight` annotations. Traefik had its own syntax. Switch controllers and you would have to rewrite everything.

Gateway API makes traffic splitting a first-class citizen. Look at how clean the HTTPRoute is:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-app-route
spec:
  parentRefs:
    - name: traefik-gateway
  hostnames:
    - "demo.homelab.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: demo-app-stable
          port: 80
          weight: 100
        - name: demo-app-canary
          port: 80
          weight: 0
```

The initial state is stable at 100%, canary at 0%. During a release, Argo Rollouts updates the weight values and Traefik applies the changes in real time. This is a standard API, not annotation magic.

---

## Demo: watching the system work

To prove this setup can actually catch problems and self-heal, I added a fault injection feature to the Demo App. Setting `errorRate: 50` in Helm Values makes the new version return HTTP 500 errors 50% of the time.

Here is what happens during a release:

1. **Git Push** -- update the image tag and push to the repository
2. **ArgoCD Sync** -- ArgoCD detects the change and syncs the new Rollout spec to the cluster
3. **Canary starts** -- Argo Rollouts spins up new Pods and updates the HTTPRoute to send 20% of traffic to them
4. **Analysis runs** -- the AnalysisTemplate queries Prometheus, which reports a success rate of around 50%
5. **Auto rollback** -- the analysis fails, Argo Rollouts immediately sets the HTTPRoute weight back to 100% stable, and marks the new version as `Degraded`

The whole thing takes about two to three minutes, with zero manual intervention. The blast radius is limited to 20% of traffic for a very short window.

Compare that to a full rollout: 100% of users affected, lasting until you manually roll back. The difference is night and day.

---

## Lessons learned

Setting this up was not exactly smooth sailing. Here are the issues that cost me the most time:

### Gateway API CRD version mismatch

Traefik's Gateway API support is evolving rapidly. Different Traefik versions support different Gateway API CRD versions. I initially installed the latest CRDs, but Traefik did not recognize them and threw cryptic errors.

The fix: check Traefik's release notes for the Gateway API version it supports, and install the matching CRDs. Do not just grab the latest.

### The PromQL empty result trap

This one is subtle. When the canary has just started and has not received any traffic yet, the PromQL query returns empty (not zero -- literally no data). Argo Rollouts interprets empty results as a failure and triggers a rollback.

In other words, your new version gets rolled back before it even serves its first request.

The fix is to add a fallback to the PromQL query:

```promql
sum(rate(http_requests_total{status!~"5.*"}[1m]))
/
sum(rate(http_requests_total[1m]))
or vector(1)
```

The `or vector(1)` means: if the preceding query returns empty, substitute 1 (i.e., 100% success rate).

### Local resource consumption

Running Traefik + ArgoCD + Argo Rollouts + Prometheus + a Demo App in a local Kind cluster eats 4-5 GB of memory just at idle. Give Docker Desktop at least 8 GB, or you will spend hours debugging OOMKilled Pods thinking it is a configuration issue.

---

## Wrapping up

With this project, my Homelab release process went from "push and pray" to "push and relax."

GitOps solved **consistency** (cluster state = Git state). Argo Rollouts solved **safety** (progressive delivery + auto rollback). Prometheus solved **observability** (data over gut feeling). Together, they form what I consider a complete GitOps release pipeline.

If you are tired of the anxiety that comes with all-at-once deployments, check out the project:

**Repository**: [canary-deployment](https://github.com/fullstackjam/canary-deployment)
