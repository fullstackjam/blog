+++
title = "我的 GitOps 金丝雀发布实践：Argo Rollouts 与 Gateway API"
date = 2025-11-27
description = "从零开始构建基于 Argo Rollouts 和 Gateway API 的金丝雀发布流水线，实现自动化流量切分、指标分析与自动回滚，告别全量发布的提心吊胆"
tags = ["gitops", "canary", "kubernetes", "argo-rollouts", "gateway-api", "traefik", "prometheus"]

[extra.comments]
issue_id = 13

[[extra.faq]]
question = "金丝雀发布和蓝绿部署有什么区别？"
answer = "蓝绿部署是维护两套完整环境，一次性切换全部流量。金丝雀发布是渐进式的，先给新版本分配少量流量（比如 5%-20%），观察指标正常后再逐步增加，风险更可控。Argo Rollouts 两种策略都支持。"

[[extra.faq]]
question = "为什么选 Gateway API 而不是 Ingress？"
answer = "Ingress 的流量切分依赖各家控制器的私有注解，写法不统一，换个控制器就得改配置。Gateway API 是 Kubernetes 官方推进的新标准，HTTPRoute 原生支持 weight 字段做流量切分，跨控制器通用。"

[[extra.faq]]
question = "Argo Rollouts 和 ArgoCD 是什么关系？"
answer = "ArgoCD 负责 GitOps 同步，确保集群状态和 Git 仓库一致。Argo Rollouts 负责发布策略，控制新版本怎么上线。两者独立但互补：ArgoCD 管'部署什么'，Rollouts 管'怎么发布'。"

[[extra.faq]]
question = "本地开发怎么测试金丝雀发布？"
answer = "可以用 Kind 或 k3d 创建本地集群，安装 Traefik + Argo Rollouts + Prometheus。不过要注意给 Docker 分配足够内存（建议 8GB+），因为组件比较多。项目仓库里有完整的本地部署指南。"

[[extra.faq]]
question = "PromQL 查询返回空值怎么办？"
answer = "这是常见坑。当没有流量时 PromQL 返回空值，Argo Rollouts 会判定为失败并触发回滚。解决方法是在查询中加 or vector(1) 兜底，让无流量时默认返回成功。"
+++

还记得我之前写的 [k8s-gitops](https://github.com/fullstackjam/k8s-gitops) 项目吗？那个项目帮我把 Homelab 的基础设施代码化了，ArgoCD 一跑，集群状态就和 Git 仓库保持一致。"怎么部署"这个问题算是解决了。

但是用了几个月之后，一个新的焦虑冒出来了：**怎么安全地更新？**

<!--more-->

每次改了代码推上去，ArgoCD 自动同步，新版本直接全量上线。我的心情可以用四个字概括——听天由命。万一代码有 Bug？万一配置写错了？所有用户（虽然主要是我的家人和朋友）瞬间受到影响，想回滚都要手忙脚乱操作一番。

这让我开始想：能不能像大厂那样，先给一小部分流量看新版本，确认没问题了再全量推广？

于是就有了这个项目：[canary-deployment](https://github.com/fullstackjam/canary-deployment)。

---

## 全量发布的问题

先说说"大爆炸"发布为什么让人不踏实。

传统的 Kubernetes Deployment 滚动更新，虽然不是瞬间切换，但新 Pod 一 Ready 就开始接流量，整个过程只有几十秒。如果新版本有问题，你发现的时候，可能已经有大量请求打到了有问题的 Pod 上。

更要命的是，Deployment 的回滚依赖的是你自己去执行 `kubectl rollout undo`，或者 ArgoCD 手动 Sync 到上一个版本。在凌晨两点被告警叫醒的时候，你确定自己能操作准确？

我想要的是这样一个流程：

{% mermaid() %}
graph LR
    A[Git Push] --> B[ArgoCD Sync]
    B --> C[创建 Canary Pod]
    C --> D[切 20% 流量]
    D --> E{指标分析}
    E -->|正常| F[切 50% 流量]
    F --> G{指标分析}
    G -->|正常| H[全量上线]
    E -->|异常| I[自动回滚]
    G -->|异常| I

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

代码推上去之后，系统自动完成"试探 -> 观察 -> 推进/止损"的完整闭环，不需要人工介入。这才是 GitOps 该有的样子。

---

## 技术选型：三驾马车

要实现上面的流程，需要三个核心组件各司其职：

**Argo Rollouts** -- 渐进式交付控制器。它扩展了 Kubernetes 原生的 Deployment，提供了 `Rollout` 资源，让你可以定义金丝雀或蓝绿发布策略。说白了，它就是那个指挥"先切多少流量、什么时候推进、什么时候回滚"的大脑。

**Gateway API + Traefik** -- 流量切分的执行者。Gateway API 是 Kubernetes 网络层的新标准（你可以理解为 Ingress 的继任者），它的 `HTTPRoute` 资源原生支持 `weight` 字段来做流量分配。Traefik 作为 Gateway Controller 负责实际执行。和以前 Ingress 时代各家靠私有注解实现流量切分不同，Gateway API 是标准化的，换控制器也不用改配置。

**Prometheus** -- 指标采集和查询。它负责回答一个关键问题："新版本到底行不行？"。Argo Rollouts 会定期向 Prometheus 发起查询，根据返回的指标数据决定继续推进还是回滚。

三者分工明确：Rollouts 做决策，Gateway API 做执行，Prometheus 做监控。

---

## 核心实现

### Rollout：重新定义"发布"

在 Argo Rollouts 的世界里，我们不再使用 `Deployment`，而是使用 `Rollout` 资源。它长得和 Deployment 几乎一样，关键区别在于多了个 `strategy` 字段：

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

这段配置定义了一个三步走的发布策略：

1. 镜像更新后，先创建新版本 Pod，把 20% 的流量切过去
2. 暂停 30 秒，这段时间 Prometheus 在持续采集指标
3. 指标正常的话，流量升到 50%，再观察 30 秒
4. 最后全量上线

注意这里有两个 Service：`demo-app-stable` 指向旧版本 Pod，`demo-app-canary` 指向新版本 Pod。Argo Rollouts 通过修改 HTTPRoute 里的 weight 来控制流量在两者之间的分配比例。

### AnalysisTemplate：让数据说话

光有流量切分只是第一步。真正的关键是：**谁来决定"继续"还是"回滚"？**

靠人盯着 Grafana 看仪表盘？凌晨三点你盯得住吗？

`AnalysisTemplate` 就是 Argo Rollouts 的自动裁判。我定义了一个基于成功率的检查：

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

这个模板告诉 Argo Rollouts：

- 每 1 分钟查一次 Prometheus
- 总共查 3 次
- 只要 HTTP 成功率低于 99%，就算失败
- 允许最多 1 次失败，超过就触发回滚

然后在 Rollout 的 steps 里引用它：

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

这样，流量切到 20% 之后，Argo Rollouts 会自动运行分析。分析通过才继续推进，失败就自动回滚。全程不需要人工干预。

### Gateway API：流量切分的标准化

以前用 Ingress 做金丝雀，Nginx 要写 `nginx.ingress.kubernetes.io/canary-weight` 注解，Traefik 又是另一套写法。换个 Ingress Controller 就得改一圈配置，痛苦不堪。

Gateway API 把流量切分变成了一等公民。看看 HTTPRoute 的配置，简洁明了：

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

初始状态 stable 100%、canary 0%。发布时 Argo Rollouts 自动修改 weight 值，Traefik 实时生效。这是标准 API，不是注解黑魔法。

---

## 演示：看系统自己干活

为了验证这套系统真的能自动止损，我在 Demo App 里加了个故障注入功能。Helm Values 里设置 `errorRate: 50`，模拟新版本有 50% 概率返回 500 错误。

发布过程是这样的：

1. **Git Push** -- 修改镜像 tag，推送到仓库
2. **ArgoCD Sync** -- ArgoCD 检测到 Git 变化，同步新的 Rollout 配置到集群
3. **金丝雀启动** -- Argo Rollouts 创建新版本 Pod，修改 HTTPRoute 将 20% 流量导向新 Pod
4. **分析运行** -- AnalysisTemplate 开始查询 Prometheus，发现成功率只有 50% 左右
5. **自动回滚** -- 分析判定失败，Argo Rollouts 立即将 HTTPRoute weight 切回 stable 100%，新版本标记为 `Degraded`

整个过程大概两三分钟，不需要任何人工操作。影响范围也被控制在了 20% 的流量内，而且持续时间很短。

对比一下全量发布：100% 的用户受影响，持续到你手动回滚为止。差距一目了然。

---

## 踩过的坑

实话说，搭建过程并不顺利。记录几个让我折腾了比较久的问题：

### Gateway API CRD 版本对齐

Traefik 对 Gateway API 的支持在快速迭代中，不同版本的 Traefik 支持的 Gateway API CRD 版本不一样。我一开始装了最新的 CRD，结果 Traefik 不认识，报了一堆看不懂的错。

解决方法：查 Traefik 的 Release Notes，确认它支持的 Gateway API 版本，然后安装对应版本的 CRD。别贪新。

### PromQL 空值陷阱

这个坑最隐蔽。当新版本刚启动、还没有流量的时候，PromQL 查询返回的是空值（不是 0，是真的没有数据）。Argo Rollouts 拿到空值后会判定分析失败，直接回滚。

也就是说，你的新版本还没接到第一个请求，就被回滚了。

解决方法是在 PromQL 里加兜底逻辑：

```promql
sum(rate(http_requests_total{status!~"5.*"}[1m]))
/
sum(rate(http_requests_total[1m]))
or vector(1)
```

`or vector(1)` 的意思是：如果前面的查询返回空值，就用 1（100% 成功率）代替。

### 本地资源消耗

在本地 Kind 集群里跑 Traefik + ArgoCD + Argo Rollouts + Prometheus + Demo App，光 idle 状态就要吃掉 4-5GB 内存。建议给 Docker Desktop 分配至少 8GB 内存，否则 Pod 会因为 OOMKill 不停重启，你还以为是配置问题。

---

## 总结

通过这个项目，我的 Homelab 发布流程从"推代码然后祈祷"变成了"推代码然后喝茶"。

GitOps 解决了**一致性**（集群状态 = Git 仓库状态），Argo Rollouts 解决了**安全性**（渐进式发布 + 自动回滚），Prometheus 解决了**可观测性**（用数据代替直觉）。三者组合，才是我心目中完整的 GitOps 发布体系。

如果你也受够了全量发布的提心吊胆，可以来我的仓库看看：

**项目地址**：[canary-deployment](https://github.com/fullstackjam/canary-deployment)
