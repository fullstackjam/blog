+++
title = "我的 GitOps 金丝雀发布实践：Argo Rollouts 与 Gateway API"
date = 2025-11-27
description = "从零开始构建基于 Argo Rollouts 和 Gateway API 的金丝雀发布流水线，实现自动化流量切分与回滚"
tags = ["gitops", "canary", "kubernetes", "argo-rollouts", "gateway-api"]
[extra.comments]
issue_id = 13
+++

还记得我之前写的 [k8s-gitops](https://github.com/fullstackjam/k8s-gitops) 项目吗？那个项目帮我把 Homelab 的基础设施代码化了，解决了"怎么部署"的问题。

但是，随着服务越来越多，我发现了一个新的痛点：**怎么安全地更新？**

每次更新服务，心里都慌慌的。直接 `kubectl apply` 或者 ArgoCD 同步之后，新版本立马全量上线。万一代码有 Bug，或者配置写错了，所有用户（其实主要是我的家人和朋友）瞬间就会受到影响。

这让我开始思考：能不能像大厂那样，先给一小部分人看新版本，没问题了再全量推广？

于是，就有了这个新项目：[canary-deployment](https://github.com/fullstackjam/canary-deployment)。

<!--more-->

---

## 为什么需要金丝雀发布？

在传统的"大爆炸"（Big Bang）发布模式下，新版本一上线就是 100% 的流量。这就好比你把一桶水直接倒进池子里，水质好坏立马影响全池。

而**金丝雀发布（Canary Deployment）**，就像是矿工带进矿井的金丝雀。我们先引入极少量的流量（比如 5%）给新版本，观察它的表现（错误率、延迟等）。
- 如果金丝雀"活"得好好的（指标正常），我们就慢慢增加流量，直到 100%。
- 如果金丝雀"挂"了（指标异常），系统自动切回旧版本，把影响范围控制在最小。

听起来很美好，对吧？但在 Kubernetes 原生对象里（Deployment/Service），想要实现这种精细的流量控制和自动化回滚，简直是噩梦。

## 技术选型：三驾马车

为了实现这个目标，我调研了一圈，最后选定了这套"三驾马车"：

1.  **Argo Rollouts**：Kubernetes 的渐进式交付控制器。它扩展了 K8s 的能力，让我们可以定义"发布策略"。
2.  **Gateway API (Traefik)**：K8s 网络的新标准。相比老旧的 Ingress，Gateway API 的 `HTTPRoute` 原生支持流量切分（Traffic Splitting），这正是金丝雀发布的核心。
3.  **Prometheus**：监控系统的标配。它负责收集指标，告诉 Argo Rollouts "新版本到底行不行"。

## 核心实现：从配置到落地

### 1. 重新定义"发布"

在 Argo Rollouts 的世界里，我们不再使用 `Deployment`，而是使用 `Rollout` 资源。

看看我的 `rollout.yaml` 配置，核心就在 `strategy` 部分：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-app
spec:
  strategy:
    canary:
      # 引用 Service，分别指向稳定版和金丝雀版
      stableService: demo-app-stable
      canaryService: demo-app-canary
      trafficRouting:
        plugins:
          # 使用标准 Gateway API 插件
          argoproj-labs/gatewayapi:
            httpRoute: demo-app-route
            namespace: default
            serviceName: demo-app-stable
      steps:
        - setWeight: 20    # 第一步：切 20% 流量给新版本
        - pause: {duration: 30s} # 暂停 30秒，让人工或自动检查
        - setWeight: 50    # 第二步：切 50% 流量
        - pause: {duration: 30s}
        - setWeight: 100   # 最后：全量上线
```

这段配置的意思是：当镜像更新时，不要直接替换所有 Pod。先给 20% 的流量，观察 30 秒；没问题再加到 50%，再观察；最后才全量。

### 2. 自动分析：让数据说话

光有流量切分还不够，关键是谁来决定"继续"还是"回滚"？靠人盯着 Grafana 看太累了。

这时候 `AnalysisTemplate` 就派上用场了。我定义了一个简单的成功率检查：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus-server.prometheus-system.svc.cluster.local:80
        query: |
          sum(rate(http_requests_total{status!~"5.*"}[1m])) / 
          sum(rate(http_requests_total[1m]))
```

Argo Rollouts 会在发布过程中不断运行这个 Prometheus 查询。只要 HTTP 请求的成功率低于 99%，发布就会立即暂停并自动回滚。

### 3. 流量管理的未来：Gateway API

以前用 Ingress 做流量切分，每家 Ingress Controller 的注解（Annotation）写法都不一样，乱得很。

现在用了 Gateway API，一切都标准化了。Argo Rollouts 只需要修改 `HTTPRoute` 里的 `weight` 字段，就能精准控制流量流向。Traefik 作为底层实现，完美支持这一标准。

## 演示时刻：见证"魔法"

为了验证这套系统，我特意在 Demo App 里加了个"坏心思"——故障注入。

我可以在 Helm Values 里设置 `errorRate: 50`，模拟新版本有 50% 的概率报错。

1.  **提交代码**：修改镜像 tag，推送到 Git。
2.  **ArgoCD 同步**：ArgoCD 发现变化，应用新的 Rollout。
3.  **金丝雀启动**：新版本 Pod 启动，Argo Rollouts 修改 HTTPRoute，将 20% 流量导向新 Pod。
4.  **自动回滚**：Prometheus 抓取到错误率飙升（因为我注入了故障），AnalysisTemplate 检查失败。
5.  **止损**：Argo Rollouts 立即将流量切回 0%，并将新版本标记为 `Degraded`。

整个过程不需要我介入，系统自己就完成了"试错-发现-止损"的闭环。这才是 GitOps 的完全体啊！

## 踩坑心得

当然，过程也不是一帆风顺的：

*   **Gateway API 版本**：Traefik 对 Gateway API 的支持还在快速迭代中，CRD 的版本要对齐，否则会报各种莫名其妙的错。
*   **Prometheus 查询**：写 PromQL 真的要小心，特别是处理"没有流量"的情况。如果查询结果为空，Argo Rollouts 可能会误判。一定要处理好 `NaN` 或空值的情况。
*   **本地测试**：在本地 Kind 集群里跑这套东西，对电脑资源是个考验。建议给 Docker 分配足够的内存。

## 总结

通过这个项目，我终于把"发布"这件事从"听天由命"变成了"运筹帷幄"。

GitOps 解决了**一致性**问题，而 Argo Rollouts + Gateway API 解决了**稳定性**问题。现在的 Homelab，不仅部署自动化了，连发布也智能化了。

如果你也想体验这种"看着系统自己干活"的快感，欢迎来我的 GitHub 仓库看看：

👉 **项目地址**：[https://github.com/fullstackjam/canary-deployment](https://github.com/fullstackjam/canary-deployment)

别忘了点个 Star 哦！🌟
