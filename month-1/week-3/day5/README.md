好的，欢迎来到第 19 天的学习！昨天我们学会了如何查看应用的“日志”，那是应用的“眼睛”。今天，我们要学习如何观察应用的“心跳和脉搏”——也就是监控与 Metrics（指标）。没有监控，你的集群就是一个黑盒子，出现问题时将无从下手。

让我们开始今天的学习，点亮你的 Kubernetes 集群！

-----

### 🎯 学习内容：理论知识篇

#### 1\. 为什么需要监控？

在生产环境中，我们需要持续回答以下问题：

  * **健康状况**：我的应用和集群现在运行正常吗？
  * **资源使用**：CPU 和内存使用率是否过高？节点上还有足够的资源来调度新的 Pod 吗？
  * **性能瓶颈**：应用的响应时间是不是变慢了？哪个服务是瓶颈？
  * **容量规划**：我是否需要增加更多的节点来应对未来的流量增长？
  * **自动化**：当负载升高时，系统能否自动扩容 Pod 数量？（这直接关系到 HPA - Horizontal Pod Autoscaler）

所有这些问题的答案，都来自于**监控指标 (Metrics)**。

#### 2\. Metrics Server：基础资源监控

Metrics Server 是 Kubernetes 中最基础的监控组件。你可以把它看作是集群的“体温计”。

  * **它的作用**: 它是一个集群范围内的聚合器，从每个节点上的 Kubelet 收集容器的**当前**资源使用情况（主要是 CPU 和内存），并通过 Kubernetes API Server 以 Metrics API 的形式暴露出来。
  * **它实现了什么**:
    1.  **`kubectl top` 命令**: 没有 Metrics Server，`kubectl top node` 和 `kubectl top pod` 命令将无法工作。
    2.  **水平 Pod 自动伸缩 (HPA)**: HPA 需要根据 Pod 的 CPU 或内存使用率来决定是否要增加或减少副本数量。它的数据来源就是 Metrics Server。
  * **它的局限性**:
      * **不存储历史数据**: 它只提供当前的资源快照，如果你想看昨天下午3点的 CPU 使用率，它做不到。
      * **指标有限**: 只提供最基础的 CPU 和内存指标，没有更详细的应用指标（如 QPS、响应时间等）。

简单来说，**Metrics Server 是 K8s 内置自动伸缩和基础资源查看的基石**，但它不是一个完整的监控解决方案。

#### 3\. Prometheus + Grafana：生产级监控黄金搭档

这是目前云原生领域监控的**事实标准**。

  * **Prometheus (普罗米修斯)**：一个开源的监控和警报系统。

      * **拉取模型 (Pull Model)**: Prometheus 会周期性地从被监控的目标（如应用、节点）暴露的 HTTP 端点上**主动拉取 (scrape)** 指标数据。
      * **时序数据库 (TSDB)**: 它将拉取到的指标数据连同时间戳一起存储在内置的高效时序数据库中，非常适合存储和查询这类“指标随时间变化”的数据。
      * **强大的查询语言 (PromQL)**: 你可以使用 PromQL 对指标数据进行丰富的查询、聚合和计算。
      * **服务发现**: 能自动发现 Kubernetes 中的监控目标，当新的 Pod 上线时，Prometheus 能自动开始监控它。
      * **告警**：内置 Alertmanager 组件，可以根据 PromQL 定义的规则，在满足条件时（如 CPU 使用率持续 5 分钟超过 80%）发送告警。

  * **Grafana (格拉法纳)**：一个开源的指标分析和可视化平台。

      * Prometheus 负责“采集和存储”，Grafana 负责“展示”。
      * 它支持包括 Prometheus 在内的多种数据源。
      * 你可以通过 Grafana 创建非常炫酷且信息丰富的仪表盘 (Dashboard)，将 Prometheus 中的原始指标数据以图表、仪表盘等形式直观地展示出来。

#### 4\. Pod 指标与集群节点指标

  * **`kubectl top` 提供的指标**：
      * `kubectl top node`: 显示每个节点的 CPU 和内存的**绝对使用量**和**占用总容量的百分比**。
      * `kubectl top pod`: 显示每个 Pod 的 CPU 和内存的**绝对使用量**。
  * **Prometheus 提供的指标 (更全面)**：
      * **节点指标**: 通过在每个节点上部署 `node-exporter`，Prometheus 可以获取到极其丰富的节点指标，如 CPU 使用情况（分用户态、内核态、iowait 等）、内存（分 used, free, buffers, cached）、磁盘 I/O、网络流量等。
      * **Pod/容器指标**: Prometheus 直接从 Kubelet 获取容器级别的 CPU、内存、文件系统、网络等指标。
      * **K8s 组件指标**: 监控 API Server, Scheduler 等核心组件的健康状况。
      * **应用业务指标**: 这是最重要的！你可以通过在你的应用代码中集成 Prometheus 客户端库，来暴露业务相关的指标（例如：API 请求总数、错误率、处理延迟等）。

#### 5\. Prometheus Operator：让 Prometheus 部署和管理更简单

在 Kubernetes 中手动部署和配置 Prometheus 体系（包括服务发现、告警规则等）是一件非常繁琐的事情。

**Prometheus Operator** 的出现就是为了解决这个问题。它利用 Kubernetes 的 [Operator 模式](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/operator/)，将对 Prometheus 的运维知识代码化，让我们能用一种“Kubernetes 原生”的方式来管理 Prometheus。

你不再需要手动修改 Prometheus 的配置文件，而是通过创建一些简单的 Kubernetes 自定义资源 (CRD)，比如：

  * `ServiceMonitor`: 定义如何监控一个 Service。只要创建了这个资源，Operator 就会自动让 Prometheus 开始监控这个 Service 关联的所有 Pod。
  * `PrometheusRule`: 定义 Prometheus 的告警和记录规则。

**简单来说，Prometheus Operator 把管理 Prometheus 的难度从“专家级”降低到了“入门级”。**

理论学习完毕，让我们开始实战，亲手触摸一下集群的“脉搏”。

-----

### 🎯 实战任务：动手操作篇

#### 第 1 步：启用 Minikube 的 Metrics Server

首先，确保你的 Minikube 正在运行。

```bash
minikube start
```

执行命令启用插件：

```bash
minikube addons enable metrics-server
```

**验证插件是否启动：**
Metrics Server 需要一点时间来启动并收集第一批数据。

```bash
# 查看 metrics-server pod 是否在 kube-system 命名空间中运行
kubectl get pods -n kube-system | grep metrics-server
```

你应该会看到一个名为 `metrics-server-xxxx` 的 Pod 处于 `Running` 状态。

**注意**：在刚启用的 1-2 分钟内，执行 `kubectl top` 命令可能会报错 `Metrics not available yet`，这是正常的，请耐心等待一下。

#### 第 2 步：使用 `kubectl top` 查看资源使用情况

1.  **查看节点 (Node) 资源使用情况：**

    ```bash
    kubectl top node
    ```

    你会看到类似下面的输出，清晰地展示了 Minikube 节点的 CPU 和内存使用情况。

    ```
    NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    minikube   263m         13%    1034Mi          53%
    ```

2.  **查看 Pod 资源使用情况：**
    默认情况下，你的 `default` 命名空间可能没有太多 Pod。我们可以先部署一个应用来观察。我们就用昨天的 Nginx Deployment。
    创建一个 `nginx-deployment.yaml` 文件：

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: my-nginx
      template:
        metadata:
          labels:
            app: my-nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.21
            # 添加资源请求，让 K8s 更好地调度，并让 top 命令显示更准确
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
    ```

    应用它：

    ```bash
    kubectl apply -f nginx-deployment.yaml
    ```

    等待 Pod 启动后，查看 Pod 的资源使用：

    ```bash
    kubectl top pod
    ```

    输出会像这样，显示每个 Nginx Pod 极低的资源占用：

    ```
    NAME                                CPU(cores)   MEMORY(bytes)
    nginx-deployment-7848d649d8-7g8ht   1m           5Mi
    nginx-deployment-7848d649d8-v9qkz   1m           5Mi
    ```

    你也可以使用 `-A` 参数查看所有命名空间中 Pod 的资源使用情况，你会发现 `kube-system` 下的系统组件占用了大部分资源。

    ```bash
    kubectl top pod -A
    ```

#### 第 3 步：了解如何部署 Prometheus Operator

如理论部分所述，我们不需要手动部署，只需要了解现在社区推荐的方式即可。目前最流行的方式是使用 **Helm** 包管理器来安装 `kube-prometheus-stack`，这是一个包含了 Prometheus Operator 以及它管理的全套监控组件的“全家桶”。

**以下命令你不需要实际执行，只需理解它们的作用即可。**

```bash
# 1. 添加 prometheus-community 的 Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 2. 更新你的本地 Helm 仓库缓存
helm repo update

# 3. (核心步骤) 使用 Helm 安装 kube-prometheus-stack
# 这会在集群中创建一个名为 "monitoring" 的命名空间，并将所有组件安装进去
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

**仅仅执行上面这一条 `helm install` 命令**，你就可以在集群中获得：

  * 一个高可用的 Prometheus 服务
  * Prometheus Operator
  * Grafana，并已内置了许多常用的 K8s 监控仪表盘
  * Alertmanager
  * Node Exporter 等各种指标采集器
  * 一套预置的通用告警规则

这就是现代 Kubernetes 生态系统的强大之处：通过社区的最佳实践，将复杂的部署流程简化为一条命令。

-----

### 🧹 清理环境

1.  **删除 Nginx 应用:**
    ```bash
    kubectl delete -f nginx-deployment.yaml
    ```
2.  **禁用 Metrics Server 插件:**
    ```bash
    minikube addons disable metrics-server
    ```
3.  **停止 Minikube:**
    ```bash
    minikube stop
    ```

-----

### 总结

太棒了，你已经完成了第 19 天的学习！今天你打开了 Kubernetes 监控的大门：

1.  **理解了基础监控**：你知道了 `Metrics Server` 是 `kubectl top` 和 `HPA` 的数据基础，但功能有限。
2.  **掌握了生产级方案**：你了解了 `Prometheus` + `Grafana` 这套黄金组合的工作原理，它是云原生监控的事实标准。
3.  **亲手实践了 `top` 命令**：你成功启用了 `metrics-server` 并使用 `kubectl top` 命令亲眼看到了节点和 Pod 的实时资源消耗。
4.  **学会了“捷径”**：你知道了 `Prometheus Operator` 和 `kube-prometheus-stack` Helm Chart 是在 K8s 上部署监控系统的最佳实践。

监控是确保服务稳定可靠的基石。有了今天的知识，你就具备了从“部署应用”到“运维应用”转变的关键视野。继续保持！