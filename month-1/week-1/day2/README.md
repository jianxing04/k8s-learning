非常好，我们来到了 Day 2！昨天的 Pod 基础打得很好，今天我们将学习更强大、更实用的资源：**Deployment**。

几乎在所有生产环境中，你都不会直接部署 Pod，而是通过 Deployment 来管理它们。让我们开始吧！

-----

### 🎯 **目标 2：理解 Deployment 如何管理 Pod，支持扩缩容和滚动更新**

-----

### **学习内容**

#### **1. Deployment 与 ReplicaSet 的关系**

还记得昨天我们说 Pod 是“房子”吗？如果白蚁（节点故障）弄塌了你的房子，那它就没了。你肯定不希望服务就此中断，你需要一个“物业管家”来确保小区里始终有足够数量的、完好的房子。

  * **ReplicaSet (副本集)**: 这就是那个“物业管家”。它的唯一职责就是**确保预定数量（replicas）的 Pod 副本一直在运行**。如果一个 Pod 挂了，ReplicaSet 会立刻新建一个来替代它，实现“自愈”能力。

  * **Deployment**: 这可以看作是“开发商/项目经理”。你不会直接去跟物业管家打交道。你会告诉项目经理：“我这个项目（比如 Nginx 服务）需要 3 个房子，房子的户型图纸是这样的（Pod 模板），以后要升级也是你来负责。”

**它们的关系是这样的：**

**Deployment (你主要操作的对象)**
└── **ReplicaSet (由 Deployment 自动创建和管理)**
├── **Pod 1**
├── **Pod 2**
└── **Pod 3**

**工作流程：**

1.  你创建一个 Deployment，在其中定义你想要的 Pod 模板（用什么镜像、开什么端口等）和副本数量（`replicas`）。
2.  Deployment 会根据你的定义，创建一个对应的 ReplicaSet。
3.  ReplicaSet 看到自己被要求“维护 3 个 Pod”，于是它就去创建 3 个 Pod。
4.  ReplicaSet 会持续监控，如果发现 Pod 少于 3 个，就立即补上。

> **核心思想**：Deployment 通过控制 ReplicaSet 来间接管理 Pod，从而提供了比 ReplicaSet 更强大的功能，比如**版本更新**和**回滚**。你只需要和 Deployment 对话，它会帮你处理所有底层细节。

-----

#### **2. 滚动更新 (Rolling Update) 和回滚 (Rollback) 机制**

这是 Deployment 最强大的功能之一：**实现服务的不停机更新**。

**滚动更新 (Rolling Update) 的过程：**

想象一下你要给正在运行的 3 个 Nginx v1.24 Pod 升级到 v1.25。Deployment 不会一次性杀掉所有旧 Pod 再启动新 Pod（这会导致服务中断），而是：

1.  创建一个新的 ReplicaSet，这个新 ReplicaSet 的目标是管理 Nginx v1.25 的 Pod。
2.  **“加一”**：先启动 1 个 v1.25 的新 Pod。
3.  等待这个新 Pod 成功启动并准备就绪。
4.  **“减一”**：当新 Pod 就绪后，再终止 1 个 v1.24 的旧 Pod。
5.  重复以上“加一减一”的过程，直到所有旧 Pod 都被新 Pod 替换。

整个过程中，始终有可用的 Pod 在提供服务，从而实现了平滑的、用户无感的更新。

**回滚机制 (Rollback)：**

如果更新到 v1.25 后发现有 Bug 怎么办？

Deployment 会记录下历史版本（也就是旧的 ReplicaSet）。当你执行回滚命令时，Deployment 只是简单地把新旧 ReplicaSet 的目标副本数调换一下：

  * 将 v1.25 的新 ReplicaSet 副本数降为 0。
  * 将 v1.24 的旧 ReplicaSet 副本数恢复到 3。

这个过程同样是滚动的，非常快速和安全。

-----

#### **3. 扩缩容：手动和自动**

  * **手动扩缩容 (Manual Scaling)**
    非常简单，就是直接修改 Deployment 配置中的 `replicas` 字段的值。比如从 3 改成 5，Deployment 会让它管理的 ReplicaSet 再创建 2 个新的 Pod。反之，从 5 改成 2，就会终止 3 个 Pod。

  * **自动扩缩容 (Horizontal Pod Autoscaler - HPA)**
    这是 Kubernetes 的高级特性，今天我们先简单了解。

    HPA 就像一个智能温控器。你可以设定一个规则，比如：“当所有 Pod 的平均 CPU 使用率超过 60% 时，就自动增加 Pod 数量；当低于 20% 时，就自动减少 Pod 数量。”

    HPA 会持续监控 Pod 的资源使用情况，并根据你设定的规则自动调整 Deployment 的 `replicas` 数量。这对于应对突发流量、节约云资源成本非常有用。我们之后的课程会深入学习它。

-----

### **实战任务**

理论已经足够，让我们马上动手实践！

#### **1. 写一个 `nginx-deployment.yaml`，指定 3 个副本**

1.  **创建 YAML 文件**：
    新建一个名为 `nginx-deployment.yaml` 的文件，内容如下：

    ```yaml
    # nginx-deployment.yaml
    apiVersion: apps/v1  # 注意：Deployment 的 apiVersion 是 apps/v1
    kind: Deployment
    metadata:
      name: my-nginx-deployment
    spec:
      replicas: 3 # 期望的 Pod 副本数量
      selector: # 选择器，告诉 Deployment 要管理哪些 Pod
        matchLabels:
          app: my-nginx
      template: # Pod 模板，这部分和我们昨天写的 Pod YAML 几乎一样
        metadata:
          labels: # Pod 的标签，必须和上面的 selector.matchLabels 一致
            app: my-nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.24 # 我们先用 1.24 版本
            ports:
            - containerPort: 80
    ```

    **和昨天的 Pod YAML 对比，关键区别：**

      * `kind` 是 `Deployment`。
      * `spec` 下多了 `replicas` 和 `selector` 字段。
      * 原来的 Pod 定义被放到了 `spec.template` 字段下。

2.  **部署 Deployment**：

    `kubectl apply -f nginx-deployment.yaml`

3.  **检查状态**：
    你可以同时查看 Deployment, ReplicaSet 和 Pod 的状态。

    `kubectl get deployment`

    ```bash
    NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
    my-nginx-deployment   3/3     3            3           20s
    ```

    `kubectl get rs` (rs 是 ReplicaSet 的缩写)

    ```bash
    NAME                             DESIRED   CURRENT   READY   AGE
    my-nginx-deployment-7f58f7dd8b   3         3         3       25s
    ```

    `kubectl get pods`

    ```bash
    NAME                                   READY   STATUS    RESTARTS   AGE
    my-nginx-deployment-7f58f7dd8b-5plg9   1/1     Running   0          30s
    my-nginx-deployment-7f58f7dd8b-9j2qf   1/1     Running   0          30s
    my-nginx-deployment-7f58f7dd8b-j4kzl   1/1     Running   0          30s
    ```

    你会看到 1 个 Deployment, 1 个 ReplicaSet, 以及由它创建的 3 个 Pod。Pod 的名字是自动生成的，格式为 `[ReplicaSet名字]-[随机字符串]`。

-----

#### **2. 修改镜像版本，观察滚动更新过程**

现在，我们要把 Nginx 从 `1.24` 升级到 `1.25`。

1.  **修改配置**：
    你可以直接修改 `nginx-deployment.yaml` 文件，把 `image: nginx:1.24` 改为 `image: nginx:1.25`，然后再次执行 `kubectl apply -f nginx-deployment.yaml`。

    不过，为了演示方便，我们用一个更快捷的命令式方法：

    `kubectl set image deployment/my-nginx-deployment nginx=nginx:1.25`

    这个命令的意思是：“设置 `my-nginx-deployment` 这个 deployment 里，名为 `nginx` 的容器，使用新镜像 `nginx:1.25`。”

2.  **立即观察！**
    马上打开**另一个终端窗口**，快速执行下面的命令，加上 `--watch` 参数来实时监控变化：

    `kubectl get pods --watch`

    你会看到一场精彩的“交接仪式”：

      * 首先，一个新的 Pod（v1.25 版本）会进入 `ContainerCreating` 状态。
      * 当新 Pod 变为 `Running` 后，一个旧的 Pod（v1.24 版本）会进入 `Terminating` 状态。
      * 这个过程会重复 3 次，直到所有 Pod 都变成了新版本。

    <!-- end list -->

    ```bash
    # 终端监控过程中的变化示例
    NAME                                   READY   STATUS              RESTARTS   AGE
    my-nginx-deployment-7f58f7dd8b-5plg9   1/1     Running             0          5m
    my-nginx-deployment-7f58f7dd8b-9j2qf   1/1     Running             0          5m
    my-nginx-deployment-7f58f7dd8b-j4kzl   1/1     Running             0          5m
    my-nginx-deployment-58548946f-abcde    0/1     ContainerCreating   0          2s   <-- 新版本 Pod 出现了
    my-nginx-deployment-7f58f7dd8b-9j2qf   1/1     Terminating         0          5m10s  <-- 旧版本 Pod 开始终止
    my-nginx-deployment-58548946f-abcde    1/1     Running             0          10s  <-- 新版本 Pod 准备就绪
    ... (这个过程会持续下去)
    ```

3.  **检查更新后的 ReplicaSet**：
    更新完成后，再执行 `kubectl get rs`。你会发现多了一个 ReplicaSet！旧的 ReplicaSet 的 `DESIRED` 副本数变成了 0，而新的 ReplicaSet 拥有 3 个副本。

    ```bash
    NAME                             DESIRED   CURRENT   READY   AGE
    my-nginx-deployment-58548946f    3         3         3       2m   <-- 新的 RS (v1.25)
    my-nginx-deployment-7f58f7dd8b   0         0         0       8m   <-- 旧的 RS (v1.24)
    ```

-----

#### **3. 用 `kubectl scale` 把副本数改成 5**

这个操作非常简单。

1.  **执行 scale 命令**：

    `kubectl scale deployment my-nginx-deployment --replicas=5`

2.  **检查 Pod 数量**：

    `kubectl get pods`

    你会立刻看到 Pod 的数量从 3 个增加到了 5 个。

    ```bash
    NAME                                   READY   STATUS    RESTARTS   AGE
    ... (原来的 3 个 Pod)
    my-nginx-deployment-58548946f-qrstx    1/1     Running   0          15s  <-- 新增的 Pod
    my-nginx-deployment-58548946f-yz123    1/1     Running   0          15s  <-- 新增的 Pod
    ```

-----

#### **4. 用 `kubectl rollout undo` 回滚版本**

假设我们发现 `1.25` 版本有严重 bug，需要立即回滚到 `1.24`。

1.  **查看历史版本**：

    `kubectl rollout history deployment my-nginx-deployment`

    你会看到每次变更的记录：

    ```bash
    REVISION  CHANGE-CAUSE
    1         <none>  # 初始创建
    2         <none>  # 更新到 1.25
    ```

2.  **执行回滚命令**：
    这个命令会回滚到上一个版本（也就是 REVISION 1）。

    `kubectl rollout undo deployment my-nginx-deployment`

3.  **再次观察**：
    再次使用 `kubectl get pods --watch`，你会看到和刚才滚动更新类似但相反的过程：v1.24 的 Pod 被创建出来，v1.25 的 Pod 被终止。

4.  **验证回滚结果**：
    回滚完成后，检查 ReplicaSet：
    `kubectl get rs`

    你会发现，旧的 `...7f58f7dd8b` (v1.24) ReplicaSet 的副本数又变回了 5，而新的 `...58548946f` (v1.25) ReplicaSet 副本数变为了 0。Deployment 成功地“切换”回了旧版本。

    同时，可以 `describe` deployment 查看镜像版本是否已经变回 `nginx:1.24`。
    `kubectl describe deployment my-nginx-deployment | grep Image`

    ```bash
    Image:        nginx:1.24
    ```

-----

### **清理环境**

删除今天的 Deployment（它会自动删除关联的 ReplicaSet 和 Pod）：

`kubectl delete deployment my-nginx-deployment`

-----

### **本日总结**

太棒了！今天你掌握了 Kubernetes 中最核心的工作负载（Workload）资源 Deployment：

1.  **理解了 Deployment -\> ReplicaSet -\> Pod 的管理模型**，知道了为什么我们不直接用 Pod。
2.  **掌握了服务“不停机”更新的精髓**：滚动更新机制。
3.  **学会了如何安全地回滚到旧版本**，为生产环境的操作提供了保障。
4.  **实践了手动扩容和缩容**，能够灵活应对负载变化。

现在我们有了一组高可用的、可自我修复、可轻松更新的 Pod 副本。但是，问题来了：这 5 个 Pod 的 IP 地址都不同，而且每次更新后 IP 都会变。我们该如何才能方便、稳定地访问它们呢？

这正是我们 Day 3 要学习的内容：**Service**！