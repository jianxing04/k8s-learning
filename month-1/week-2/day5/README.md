欢迎来到第 12 天的学习！

我们已经掌握了如何在 Kubernetes 中部署、管理、暴露和调度各种应用。但还有一个非常关键的生产实践问题没有解决：**资源管理**。

在一个共享的集群中：

  * 如果一个 Pod 写了内存泄漏的代码，它会不会耗尽整个节点的内存，导致其他 Pod 甚至节点本身崩溃？
  * 如果一个 Pod 突然遇到流量洪峰，开始疯狂消耗 CPU，会不会影响到邻居 Pod 的性能？
  * 我们如何确保每个 Pod 都能获得它正常运行所需的最小资源？
  * 我们能否让应用在流量高峰时自动增加 Pod 数量，在流量低谷时自动减少，从而节约成本？

今天，我们就来学习解决这些问题的核心工具：**资源配额 (Requests & Limits)** 和 **水平 Pod 自动伸缩器 (HPA)**。

-----

### 🎯 **目标 12：理解资源配额与自动扩缩容**

-----

### **学习内容**

#### **1. Pod 的 `resources.requests` 与 `resources.limits`**

这是在 Pod 定义中对容器资源进行约束的两个核心字段。它们都属于 `spec.containers.resources`。

  * **`resources.requests` (资源请求)**

      * **是什么**：一个容器**保证可以获得的最小资源量**。
      * **作用**：主要用于**调度**。`kube-scheduler` 在选择节点时，会检查节点的剩余资源是否能满足 Pod 中所有容器的 `requests` 总和。如果找不到任何节点能满足，Pod 将会一直处于 `Pending` 状态。
      * **比喻**：就像你预订机票时，`requests` 就是你**确定要购买的座位**。航空公司（Kubernetes）必须保证你有这个座位，否则你根本上不了飞机（Pod 无法调度）。

  * **`resources.limits` (资源限制)**

      * **是什么**：一个容器**允许使用的资源上限**。
      * **作用**：主要用于**运行时资源约束**，由节点上的 `kubelet` 强制执行。
      * **对于 CPU**：如果容器使用的 CPU 超过了 `limits`，它会被**限流 (Throttled)**，性能会下降，但不会被杀死。
      * **对于 Memory**：**非常重要！** 如果容器使用的内存超过了 `limits`，它会被系统**杀死并重启**，这就是著名的 **OOMKilled (Out of Memory Killed)**。
      * **比喻**：`limits` 就像是你座位的**空间边界**。你的胳膊（CPU 使用）可以偶尔伸出去一点，但会被乘务员提醒（被限流）。但如果你的行李（内存使用）占用了过道，超出了允许的范围，你就会被“请下飞机”（被 OOMKilled）。

#### **2. 什么是 CPU / 内存配额**

  * **CPU 单位**

      * CPU 在 Kubernetes 中是一个绝对单位，`1` 代表 1 个 CPU 核心 (1 vCPU, 1 Core)。
      * 你可以使用毫核（millicores）来表示不足一个核心的单位，写作 `m`。`500m` 就是 0.5 个核心，`100m` 就是 0.1 个核心。

  * **内存单位**

      * 内存的单位是字节。通常使用国际单位制（SI）的后缀：`E`, `P`, `T`, `G`, `M`, `K`。
      * 更推荐使用二进制单位：`Ei`, `Pi`, `Ti`, `Gi`, `Mi`, `Ki` (例如，1 `Mi` = 1,048,576 字节)。

**Kubernetes 服务质量 (QoS) 等级**
根据 `requests` 和 `limits` 的设置，Kubernetes 会给 Pod 划分三个 QoS 等级：

1.  **Guaranteed (有保证的)**: Pod 中**每个**容器都同时设置了 `requests` 和 `limits`，并且 CPU 和内存的 `requests` 与 `limits` **完全相等**。这是最高优先级，最不容易被系统在资源紧张时杀死。
2.  **Burstable (可突发的)**: Pod 至少有一个容器设置了 `requests` 但 `limits` 更高，或者只设置了 `requests`。这是中等优先级。
3.  **BestEffort (尽力而为的)**: Pod 中没有任何容器设置 `requests` 或 `limits`。这是最低优先级，在节点资源耗尽时，这些 Pod 最先被杀死。

> **最佳实践**：为生产环境中的所有 Pod 都设置 `requests` 和 `limits`，让它们至少是 `Burstable`，最好是 `Guaranteed`，以保证服务的稳定性。

#### **3. Horizontal Pod Autoscaler (HPA)**

  * **是什么**：HPA 是 Kubernetes 的一个内置控制器，它可以根据 CPU 利用率、内存使用率或其他自定义指标，**自动地调整** Deployment、StatefulSet 等工作负载的 **Pod 副本数量**。

  * **工作原理**：

    1.  HPA 会通过一个名为 **Metrics Server** 的组件，定期（默认 15 秒）获取它所监控的所有 Pod 的资源使用情况。
    2.  它会将当前所有副本的**平均**指标值与你在 HPA 中设定的**目标值**进行比较。
    3.  根据一个公式 `期望副本数 = ceil[当前副本数 * ( 当前指标 / 目标指标 )]` 来计算出新的副本数量。
    4.  如果计算出的期望副本数与当前不同，HPA 就会更新 Deployment 的 `replicas` 字段，触发 Deployment 进行扩容或缩容。

  * **重要前提**：

    1.  **必须安装 Metrics Server**：没有它，HPA 就无法获取 Pod 的指标，也就无法工作。
    2.  **必须设置 `requests`**：HPA 计算 CPU/内存**利用率**的分母，就是你在 Pod 中设置的 `requests` 值。例如，`requests.cpu` 设为 `200m`，HPA 目标设为 50%，那么当 Pod 的实际 CPU 使用量超过 `100m` 时，HPA 就会开始考虑扩容。**如果不设置 requests，HPA 将无法工作**。

-----

### **实战任务**

#### **任务 1 & 2：部署一个带资源配额的 Nginx Deployment**

1.  **编写 `nginx-resources-deployment.yaml` 文件**:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-resources-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-hpa
      template:
        metadata:
          labels:
            app: nginx-hpa
        spec:
          containers:
          - name: nginx
            image: nginx:1.24
            ports:
            - containerPort: 80
            # --- 关键部分：资源配额 ---
            resources:
              requests:
                cpu: "200m"
                memory: "64Mi"
              limits:
                cpu: "400m"
                memory: "128Mi"
    ```
2.  **部署它**: `kubectl apply -f nginx-resources-deployment.yaml`
3.  **创建 Service** (为了后面能方便地产生流量):
    新建 `nginx-service.yaml`
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-hpa-svc
    spec:
      selector:
        app: nginx-hpa
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    ```
    部署 Service: `kubectl apply -f nginx-service.yaml`

#### **任务 3：启用 Metrics Server**

1.  在 Minikube 环境中，只需一条命令即可：
    `minikube addons enable metrics-server`

2.  等待一到两分钟，让它启动并开始收集数据。然后通过以下命令验证：
    `kubectl top pods`
    `kubectl top nodes`
    如果你能看到 Pod 和 Node 的 CPU/内存使用情况，说明 Metrics Server 已经正常工作。

#### **任务 4：创建 HPA 并触发自动扩缩容**

1.  **创建 HPA 资源**:
    我们可以用一条 `kubectl` 命令快速创建 HPA，而无需写 YAML。

    `kubectl autoscale deployment nginx-resources-deployment --cpu-percent=50 --min=1 --max=5`

      * `--cpu-percent=50`：目标 CPU 利用率是 50% (相对于 `requests.cpu` 的 `200m`，也就是 `100m`)。
      * `--min=1`：最少 1 个副本。
      * `--max=5`：最多 5 个副本。

2.  **观察 HPA 状态**:
    打开一个终端，持续监控 HPA 的状态：
    `kubectl get hpa --watch`
    初始状态下，`TARGETS` 可能是 `<unknown>/50%` 或 `0%/50%`，`REPLICAS` 是 1。

3.  **制造压力，触发扩容**！

      * 打开**另一个**终端，我们来创建一个“压力产生器” Pod，它会无限循环地访问我们的 Nginx 服务。
        `kubectl run -it --rm load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://nginx-hpa-svc; done"`

      * **观察变化**：
        回到监控 HPA 的终端，奇妙的事情发生了！

          * `TARGETS` 列的 CPU 使用率会迅速飙升，远超 50% (例如 `250%/50%`)。
          * `REPLICAS` 列的数字会开始变化，从 1 变成 2，然后 3、4、5...
          * 你也可以同时打开第三个终端 `kubectl get pods -w`，会看到新的 Nginx Pod 被一个个地创建出来。

4.  **撤销压力，触发缩容**！

      * 在“压力产生器”的终端按 `Ctrl + C`，停止发送请求。
      * **继续观察 HPA**：
          * `TARGETS` 列的 CPU 使用率会慢慢降下来，回到 `0%/50%`。
          * 等待几分钟（HPA 有一个默认的 5 分钟缩容冷却期，防止因流量抖动而频繁缩容），你会看到 `REPLICAS` 的数量会从 5 慢慢地降回到 `min` 值的 1。

-----

### **清理环境**

```bash
# 停止压力产生器（如果它还在运行）
# 删除 HPA
kubectl delete hpa nginx-resources-deployment
# 删除 Service
kubectl delete service nginx-hpa-svc
# 删除 Deployment
kubectl delete deployment nginx-resources-deployment
# (可选) 禁用 metrics-server
# minikube addons disable metrics-server
```

-----

### **本日总结**

今天你掌握了保障 Kubernetes 集群稳定性和弹性的两大支柱：

1.  **理解了 `requests` 和 `limits` 的重要性**：`requests` 保证了 Pod 的调度和最低运行资源，`limits` 防止了单个 Pod 耗尽节点资源，它们共同决定了 Pod 的 QoS 等级。
2.  **掌握了 HPA 的工作原理和基本用法**：通过设置合理的扩缩容策略，可以让应用从容应对流量变化，既保证了服务质量，又避免了资源浪费。
3.  **深刻理解到，要让 HPA 正常工作，必须为 Pod 设置 `requests`**。

你现在不仅知道如何运行应用，更知道如何让它们在共享环境中**稳定、高效、且富有弹性**地运行。这是向着专业 K8s 运维管理迈出的一大步！