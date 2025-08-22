好的，我们正式进入第 11 天的学习！

在前十天的课程中，我们主要关注的是“**部署什么**”（Pod、Deployment、StatefulSet）和“**如何访问**”（Service、Ingress）。我们一直让 Kubernetes 自动决定把 Pod 放在哪个可用的节点（Node）上。

但在真实的生产环境中，集群往往是**异构 (Heterogeneous)** 的：

  * 有些节点是普通服务器。
  * 有些节点配备了高性能的 SSD 硬盘。
  * 有些节点拥有昂贵的 GPU 显卡。
  * 有些节点位于特定的物理机架或可用区。

今天，我们就要学习如何扮演“指挥官”的角色，告诉 Kubernetes 的**调度器 (Scheduler)**，我们希望把特定的 Pod **部署到哪里**。

-----

### 🎯 **目标 11：理解 Pod 是怎么调度到节点上的**

-----

### **学习内容**

#### **1. Kubernetes 调度流程 (Scheduling Process)**

当你 `kubectl apply` 一个 Pod 的 YAML 文件后，这个 Pod 最初处于 `Pending` 状态。此时，集群里一个名为 **kube-scheduler** 的核心组件会为它寻找一个最合适的“家”（Node）。这个过程主要分两步：

1.  **过滤 (Filtering)**

      * **目的**：筛选出所有**不能**运行该 Pod 的节点，得到一个候选节点列表。
      * **过程**：调度器会像一个严格的考官，用一系列规则（称为“断言/Predicates”）去检查每个节点。任何不满足条件的节点都会被直接淘汰。
      * **过滤规则示例**：
          * 节点资源是否足够（CPU、内存）？
          * Pod 请求的端口在节点上是否已被占用？
          * 节点是否满足 Pod 指定的标签要求 (`nodeSelector`)？
          * 节点是否有“污点”(Taint)，而 Pod 是否“容忍”(Toleration) 这个污点？

2.  **打分 (Scoring)**

      * **目的**：从候选节点列表中，选出**最适合**的一个。
      * **过程**：调度器会给所有通过过滤阶段的候选节点打分。分数最高的节点胜出，Pod 就会被调度到该节点上。
      * **打分规则示例**：
          * **资源最少使用优先**：倾向于将 Pod 部署到 CPU 和内存使用率较低的节点，以实现负载均衡。
          * **镜像亲和性**：如果一个节点上已经有了 Pod 所需的容器镜像，那么它的得分会更高（因为可以节省拉取镜像的时间）。
          * **亲和性/反亲和性规则**：根据 Pod 的 `Affinity` 设置来打分。

#### **2. `NodeSelector`, `NodeAffinity`, `Taints & Tolerations`**

这是我们用来影响调度过程的三种主要工具，它们的表达能力和使用场景各有不同。

  * **`NodeSelector` (节点选择器)**

      * **是什么**：最简单、最直接的节点指定方式。
      * **工作方式**：在 Pod 的 `spec` 中指定一个 `nodeSelector` 字段，它是一个键值对的 map。调度器只会将该 Pod 调度到**同时拥有所有这些标签 (Label)** 的节点上。
      * **比喻**：就像是租房时说：“我**只**考虑带‘独立卫浴’**并且**是‘朝南’的房间。” 条件是写死的，必须完全满足。

  * **`NodeAffinity` (节点亲和性)**

      * **是什么**：`NodeSelector` 的升级版，提供了更强大、更灵活的节点选择能力。
      * **工作方式**：它支持更丰富的匹配规则（如 `In`, `NotIn`, `Exists`, `DoesNotExist` 等），并且分为两种类型：
        1.  **硬亲和性 (`requiredDuringSchedulingIgnoredDuringExecution`)**: **必须满足**。规则和 `NodeSelector` 类似，如果找不到满足条件的节点，Pod 就会一直处于 `Pending` 状态。
        2.  **软亲和性 (`preferredDuringSchedulingIgnoredDuringExecution`)**: **倾向于满足**。调度器会**尽量**寻找满足条件的节点，但如果找不到，Pod 仍然会被调度到其他不满足条件的节点上。你可以给每个偏好设置一个权重，权重越高的偏好得分越高。
      * **比喻**：
          * **硬亲和性**：“我租房**必须**在‘一号线’**或**‘二号线’地铁沿线。” (使用 `In` 操作符)
          * **软亲和性**：“我**最好**能租到一个带‘阳台’的房间，如果没有，其他合适的也行。”

  * **`Taints & Tolerations` (污点与容忍)**

      * **是什么**：这是一种**反向**的调度机制。亲和性是 Pod “吸引”到 Node 上，而污点是 Node “排斥” Pod。
      * **工作方式**：
          * **Taint (污点)**: 你给一个 **Node** 打上“污点”，这个污点会拒绝所有不能“容忍”它的 Pod 被调度上来。
          * **Toleration (容忍)**: 你在 **Pod** 上设置“容忍”，声明它可以容忍某些污点，这样它就能够被调度到带有相应污点的节点上。
      * **比喻**：房东（Node）在门上贴了一张条：“此房正在装修（Taint），闲人免入！”。只有持有特殊通行证（Toleration）的装修工人（Pod）才能进入。
      * **典型用途**：
          * **专用节点**：给 GPU 节点打上 `gpu=true:NoSchedule` 的污点，这样只有明确需要 GPU 的 Pod（并设置了相应容忍）才会被调度上去，防止普通应用占用昂贵的 GPU 资源。
          * **节点维护**：当管理员要维护一个节点时，可以先给它加上污点，阻止新的 Pod 调度上来，然后再驱逐已有 Pod。

#### **3. Pod 优先级 (Pod Priority)**

  * **是什么**：一个告诉调度器“这个 Pod 比那个 Pod 更重要”的机制。
  * **工作方式**：通过 `PriorityClass` 资源定义不同的优先级。高优先级的 Pod 在调度时会排在前面。如果集群资源不足，高优先级的 Pod 甚至可以**抢占 (Preemption)** 节点上已经运行的低优先级 Pod，把它们“踢走”来为自己腾出空间。
  * **典型用途**：确保核心系统组件（如监控、日志代理）的运行优先级高于普通的业务应用。

-----

### **实战任务**

**环境说明**：由于我们使用的是单节点的 Minikube 环境，我们无法真正“选择”节点。但是，我们可以通过设置一个**不存在的**标签，来观察 Pod 是否会因为不满足条件而卡在 `Pending` 状态，从而验证调度机制是生效的。

#### **任务 1：给 Node 打 Label**

1.  **查看当前节点**：
    `kubectl get nodes`
    你会看到一个名为 `minikube` 的节点。

2.  **给 minikube 节点打上标签**：
    我们将给它打上一个 `disktype=ssd` 的标签，假装它拥有一块 SSD 硬盘。
    `kubectl label node minikube disktype=ssd`

3.  **验证标签**：
    `kubectl get nodes --show-labels`
    在输出的 `LABELS` 列中，仔细查找，你会看到 `disktype=ssd` 已经成功添加。

    > **清理提示**：未来如果想移除这个标签，可以执行 `kubectl label node minikube disktype-`。

-----

#### **任务 2：使用 `nodeSelector` 调度 Pod**

1.  **编写 `nodeselector-pod.yaml` 文件**:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodeselector-pod
    spec:
      containers:
      - name: nginx
        image: nginx
      # --- 关键部分 ---
      nodeSelector:
        disktype: ssd
    ```

    这个 Pod 声明了它“只去”带有 `disktype=ssd` 标签的节点。

2.  **部署并验证 (成功案例)**：
    `kubectl apply -f nodeselector-pod.yaml`
    `kubectl get pods -o wide`
    因为我们的 `minikube` 节点正好有这个标签，所以 Pod 会立刻被成功调度，并进入 `Running` 状态。

3.  **制造失败案例来加深理解**：

      * 先删除刚才的 Pod: `kubectl delete pod nodeselector-pod`
      * 修改 `nodeselector-pod.yaml`，将 `ssd` 改为一个不存在的标签值，比如 `hdd`。
        ```yaml
        nodeSelector:
          disktype: hdd
        ```
      * 再次部署: `kubectl apply -f nodeselector-pod.yaml`
      * 查看状态: `kubectl get pods`，你会发现 `nodeselector-pod` 一直卡在 **`Pending`** 状态！
      * 查看原因: `kubectl describe pod nodeselector-pod`
        在输出的 `Events` 部分，你会看到一条非常明确的消息，类似：
        `Warning  FailedScheduling  ... 0/1 nodes are available: 1 node(s) didn't match node selector.`
        这完美地证明了 `nodeSelector` 的强制约束作用。

-----

#### **任务 3：使用 `nodeAffinity` 调度 Pod**

1.  **编写 `nodeaffinity-pod.yaml` 文件**:
    我们将使用“硬亲和性”，要求节点必须有一个 `disktype` 标签，并且其值是 `ssd` 或 `nvme` 中的一个。

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodeaffinity-pod
    spec:
      containers:
      - name: nginx
        image: nginx
      # --- 关键部分 ---
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: # 硬亲和性
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In # 操作符：In (值在列表中)
                values:
                - ssd
                - nvme
    ```

2.  **部署并验证**：

      * 先删除上一步失败的 Pod: `kubectl delete pod nodeselector-pod`
      * 部署新 Pod: `kubectl apply -f nodeaffinity-pod.yaml`
      * 查看状态: `kubectl get pods -o wide`
        因为 `minikube` 节点的 `disktype` 是 `ssd`，满足 `In [ssd, nvme]` 这个条件，所以 Pod 会被成功调度。这证明了 `nodeAffinity` 同样可以实现强制约束，并且表达能力更强。

-----

### **清理环境**

```bash
kubectl delete pod nodeaffinity-pod
# 如果之前那个 pending 的 pod 还在，也一并删除
kubectl delete pod nodeselector-pod
# 移除我们添加的标签
kubectl label node minikube disktype-
```

-----

### **本日总结**

今天，你掌握了 Kubernetes 调度器的基本工作原理，并学会了如何影响它的决策：

1.  **理解了调度分为“过滤”和“打分”两个阶段**。
2.  **掌握了 `NodeSelector`**：一种简单直接的、基于标签的硬性调度约束。
3.  **掌握了 `NodeAffinity`**：一种更强大、更具表达力的调度约束，支持软硬两种亲和性。
4.  **理解了 `Taints & Tolerations` 的概念**：一种反向的、由节点排斥 Pod 的调度控制机制。

现在，你不仅能决定要部署什么应用，还能精细地控制它们应该运行在哪些具有特定能力的节点上，这对于管理一个复杂的、异构的生产集群至关重要。