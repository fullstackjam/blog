+++
title = "我的 Homelab 如何践行现代 DevOps 核心理念"
date = 2025-09-25
description = "三年 Homelab 折腾下来的总结：从雪花服务器到 GitOps，从手动部署到自动监控，一路是怎么演进的。附架构图和踩坑记录。"
tags = ["homelab", "devops", "kubernetes", "gitops", "自动化"]

[extra.comments]
issue_id = 10

[[extra.faq]]
question = "搭建 Homelab 需要什么硬件？"
answer = "不用很贵的设备。几台旧笔记本、迷你主机（比如 Intel NUC）或者二手服务器都行。我自己就是从几台破服务器起步的，先跑起来，硬件慢慢换。"

[[extra.faq]]
question = "Homelab 用 Kubernetes 会不会太重了？"
answer = "只跑几个容器确实用不上 K8s，Docker Compose 就够。但想系统学云原生，K8s 绕不开。而且服务一多，它的声明式管理和自动调度反而比手动省心。"

[[extra.faq]]
question = "GitOps 和传统 CI/CD 有什么区别？"
answer = "传统 CI/CD 是 push：CI 构建完直接推到服务器。GitOps 是 pull：集群里的 ArgoCD 一直盯着 Git 仓库，发现变化就自动同步。好处是 Git 成了唯一的真相来源，改了什么都有记录，回滚就是 git revert。"

[[extra.faq]]
question = "Homelab 对职业发展有帮助吗？"
answer = "帮助挺大。面试能聊真踩过的坑，而不是背概念。工作里碰到网络策略、存储卷、资源限制这些，因为在 Homelab 见过，定位快很多。最值钱的是那套 DevOps 思维方式，比会用某个工具管用。"

[[extra.faq]]
question = "从零开始搭 Homelab，推荐什么技术栈？"
answer = "建议分步走：先用 Ansible 管机器配置，把基础设施代码化；再上 K8s + ArgoCD，做 GitOps 自动部署；最后加 Prometheus + Grafana 做监控。每步先跑通再进下一步，别一口气全上。"
+++

三年前折腾 [k8s-gitops](https://github.com/fullstackjam/k8s-gitops) 的时候，我只是想搭个能跑的 K8s 环境玩玩。没想到这一路折腾下来，竟然把企业级的 DevOps 实践都给搞全了。

从几台破服务器到现在这套还算能打的集群，踩的坑、走的弯路都留在了这个项目里。回头看，DevOps 那点门道，不知不觉就摸得差不多了。

<!--more-->

## 为什么要折腾 Homelab？

说实话，一开始就是纯粹的技术好奇心在作怪。

公司里用的那套 K8s 环境，我只能看不能摸，想试点新功能都要排期。云服务虽然方便，但费用高不说，还有各种限制。想要真正理解这些技术的本质，还得自己动手搭一套。

最关键的是，看再多文档，不如自己踩一遍坑。

刚开始的时候，我连 Pod 和 Service 的区别都搞不清楚。网上教程看了一堆，到实际操作的时候还是一头雾水。但是当你自己从零开始搭建，每一个配置文件都要自己写，每一个网络问题都要自己解决的时候，这些概念就慢慢清晰起来了。

## 三年演进全景

先上一张图，看看这三年我的 Homelab 是怎么一步步进化的：

{% mermaid() %}
graph LR
    subgraph "第一阶段：手工时代"
        A[手动 SSH 配置] --> B[雪花服务器]
        B --> C[手动部署应用]
    end

    subgraph "第二阶段：自动化"
        D[Ansible 剧本] --> E[标准化节点]
        E --> F[K8s 集群]
    end

    subgraph "第三阶段：GitOps"
        G[Git 仓库] --> H[ArgoCD 同步]
        H --> I[自动部署]
        I --> J[Prometheus 监控]
        J --> K[Grafana 面板]
    end

    C -.->|"受不了了"| D
    F -.->|"手动部署太痛苦"| G

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

每个阶段都是被上一个阶段的痛点逼出来的。下面展开讲。

## 第一步：告别"雪花服务器"

### 问题：每台机器都是独一无二的"艺术品"

最开始的时候，我的每台机器都是手工配置的。这台装了 Docker 20.10，那台装了 Docker 24.0。这台的 iptables 规则是半夜调试的时候加的，那台的 `/etc/hosts` 里有一堆不知道干嘛的条目。配置文件散落各处，版本也不统一。

这就是经典的"雪花服务器"——每一台都独一无二，没人敢动，出了问题也没人能完全复现。

一旦某台机器挂了，基本就是重装系统的节奏。而且重装完之后，你根本不记得之前到底改了哪些配置。

### 解决方案：Infrastructure as Code

后来实在受不了了，开始用 **Ansible** 来管理。从操作系统安装到软件配置，全部写成剧本（playbook）。

```yaml
# 节点初始化剧本（简化版）
- hosts: k8s_nodes
  tasks:
    - name: 更新系统包
      apt:
        update_cache: yes
        upgrade: dist

    - name: 安装基础依赖
      apt:
        name:
          - curl
          - apt-transport-https
          - ca-certificates
        state: present

    - name: 配置内核参数
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
      loop:
        - { key: net.bridge.bridge-nf-call-iptables, value: "1" }
        - { key: net.ipv4.ip_forward, value: "1" }

    - name: 安装 containerd
      apt:
        name: containerd
        state: present
```

现在想重建整个集群，一行命令搞定：

```bash
ansible-playbook -i inventory/hosts site.yml
```

5 分钟内集群重新上线。更重要的是，每台机器的配置都是一模一样的，不再有"这台机器比较特殊"的说法。

把基础设施变成代码以后，最大的变化是不再怕搞坏环境了——随时能重来，代码都在 Git 里，谁改了什么一目了然。

## 第二步：GitOps 救了我的命

### 问题：手动部署是噩梦

K8s 集群搭好了，应用也跑起来了，但是每次更新都要 SSH 到服务器上，手工执行一堆命令：

```bash
# 每次部署的"标准流程"
ssh master-node
cd /home/user/manifests
git pull
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app
# 祈祷不出问题...
```

经常搞忘了某个步骤，导致服务挂掉。更可怕的是，完全不知道线上跑的是哪个版本的代码。有一次排查了两个小时的 bug，最后发现是因为线上跑的还是三天前的老版本——我以为部署了，其实忘了。

### 解决方案：ArgoCD + 声明式配置

引入 **ArgoCD** 之后，世界瞬间清净了。

原理很简单：ArgoCD 安装在集群里，持续监听 Git 仓库。你把想要的状态写成 YAML 推到 Git，ArgoCD 自动检测变化，把集群状态同步到和 Git 一致。

```yaml
# ArgoCD Application 定义
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

现在部署就是 Git push 一下，剩下的全自动。想回滚？`git revert` 然后 push，ArgoCD 自动把集群恢复到之前的状态。每次部署都有完整的 Git 记录，再也不用担心"昨天到底改了什么"这种问题。

我还给不同的应用设了不同的同步策略：

- **基础设施组件**（cert-manager、ingress-nginx）：手动同步，改之前先在 staging 验证
- **自己的应用**（博客、API）：自动同步，推了就上
- **第三方服务**（Grafana、Prometheus）：自动同步 + selfHeal，防止手动改配置导致漂移

部署变成 Git 操作之后，运维的压力一下小了一半。不用记命令，不用怕手抖，出问题 `git log` 一看就知道。

## 第三步：可观测性让我睡得安稳

### 问题：系统是个黑盒

以前系统出问题，基本靠蒙。CPU 高了？不知道。内存不够了？也不知道。直到用户投诉了才发现服务挂了。

有一次我的博客挂了整整两天我都没发现，因为我自己也不天天去看。直到一个朋友告诉我"你博客打不开了"，我才知道。

### 解决方案：Prometheus + Grafana + 告警

搭建了完整的监控体系之后，整个系统变得"透明"了：

**Prometheus** 负责采集指标——每个节点的 CPU、内存、磁盘，每个 Pod 的资源使用，每个 Service 的请求延迟和错误率。

**Grafana** 负责可视化——哪个服务消耗资源多，哪台机器负载高，一眼就能看出来。我给不同的维度做了不同的 Dashboard：

- **集群总览**：节点状态、资源使用率、Pod 数量
- **应用监控**：每个服务的 CPU/内存趋势、重启次数
- **网络监控**：ingress 流量、请求延迟分布

最关键的是**告警**。配了 AlertManager，下面这些情况会自动发通知：

```yaml
# 告警规则示例
groups:
  - name: node-alerts
    rules:
      - alert: NodeHighCPU
        expr: node_cpu_seconds_total > 0.85
        for: 5m
        annotations:
          summary: "节点 CPU 使用率超过 85%"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        annotations:
          summary: "Pod 反复重启"

      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.15
        for: 10m
        annotations:
          summary: "磁盘剩余空间不足 15%"
```

现在，问题发生之前我就能收到告警。上次有个节点的磁盘快满了，提前一天收到通知，清理了日志就没事了。要是以前，等到磁盘真满了，整个节点上的 Pod 全得挂。

监控这东西不是拿来炫技的，搞它就是为了晚上能睡安稳。

## 额外收获：这些 DevOps 理念改变了我的思维方式

三年搞下来，我发现这个项目最大的价值不是那些酷炫的技术栈，而是让我真正理解了什么是 DevOps。

以前我觉得 DevOps 就是一堆工具的组合，学会了 Docker、K8s、CI/CD 就算掌握了。实际搞下来才发现，它更像一种习惯：能自动化的就别手动，Ansible 管配置、ArgoCD 管部署、Renovate 管依赖更新；没有指标的服务等于在裸奔，能监控的尽量监控；配置别靠口头传，全扔进 Git，改了什么、谁改的一目了然；环境别只有一个人会搭，任何人拿到代码都能重建。

这些东西一旦成了习惯，看技术问题的角度就不一样了。现在写任何东西，第一反应都是：这能自动化吗？有监控吗？配置进 Git 了吗？

## 对职业发展的影响

更实际的是，这个项目确实帮了我的职业发展大忙。

面试的时候，当别人还在背概念的时候，我可以很自然地聊起实际遇到的问题和解决方案。什么网络策略为什么要设 `policyTypes`、PV 和 PVC 的绑定为什么会卡住、资源 limit 设太低会触发 OOMKill——这些都不再是抽象的概念，而是真实踩过的坑。

现在工作中遇到类似的问题，基本都能快速定位和解决。因为在 Homelab 里都遇到过类似的场景。

## 给想入坑的人的建议

如果你也想搞个 Homelab，我的经验总结成几句话：

**别一开始就搞得太复杂。** 我最初就犯了这个错误，想一口气把 K8s、Istio、Vault、ArgoCD 全上，结果搞得乱七八糟。还是得一步一步来，先把基础打牢。建议顺序：Docker → K8s → GitOps → 监控 → 其他。

**选择一个具体的项目深入下去。** 不要今天学 Docker，明天学 K8s，后天又去搞 Terraform。选定一个方向，死磕到底。我就是围绕"在家跑一个完整的 K8s 集群"这个目标一路走过来的。

**记录你踩过的每一个坑。** 我现在还经常翻自己以前的笔记，很多问题的解决方案都在里面。而且这些记录就是你最好的面试素材。

**别害怕搞坏东西。** 反正是自己的环境，坏了就重来。我这三年里重装系统不下二十次，每次都有新的收获。用了 Ansible 之后，重装也不心疼了——一条命令就恢复到之前的状态。

**别追求完美，先跑起来再说。** 我见过太多人纠结于架构设计，结果一直停留在纸面上。先搞一个 single-node 的 K8s，把东西跑起来，遇到问题再改。

**加入社区。** K8s 社区里有很多大神，多交流能少走不少弯路。GitHub 上搜 "k8s-gitops" 或 "homelab"，能找到很多优秀的参考项目。

## 最后

[k8s-gitops](https://github.com/fullstackjam/k8s-gitops) 这个项目让我信了一件事：个人项目也能有企业级的质量，关键不在规模，在理念有没有真的落到地上。

Infrastructure as Code、GitOps、可观测性这些东西，在家里这套小集群上一样跑得起来，而且学到的直接能用到工作里。

想搞 Homelab 的话别纠结，找台旧电脑就能开始。

**项目地址**：[https://github.com/fullstackjam/k8s-gitops](https://github.com/fullstackjam/k8s-gitops)

**文档地址**：[https://k8s-gitops.fullstackjam.com](https://k8s-gitops.fullstackjam.com)

有问题欢迎来 GitHub 上讨论，也欢迎 Star 支持一下。
