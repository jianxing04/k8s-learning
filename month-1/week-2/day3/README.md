很好，我们来到了第十天的学习！

在上两天的课程中，我们深入了解了 Kubernetes 的配置和存储。你已经掌握了如何使用 PVC 为 Pod 提供持久化存储，这是今天学习内容的关键前置知识。

我们之前学习的 Deployment 非常适合无状态应用（Stateless Applications）。它把所有 Pod 都看作是完全一样、可以随意替换的“牛”（Cattle）。但如果我们要部署的是数据库集群、消息队列集群这类**有状态应用 (Stateful Applications)** 呢？在这些场景中，每个 Pod 实例都是独一无二的“宠物”（Pets），它们有自己的身份、自己的数据，不能被随意替换。

今天，我们就来学习专门为这类“宠物”应用设计的控制器：**StatefulSet**。

-----

### 🎯 **目标 10：理解有状态服务的部署方式**

-----

### **学习内容**

#### **1. Deployment vs. StatefulSet**

这是理解 StatefulSet 的核心。让我们用一个表格来清晰地对比它们：

| 特性 (Feature) | Deployment | StatefulSet |
| :--- | :--- | :--- |
| **应用类型** | **无状态 (Stateless)** | **有状态 (Stateful)** |
| **Pod 身份** | 临时的、可替换的 (Cattle) | **稳定的、唯一的 (Pets)** |
| **Pod 命名** | 随机生成 (`-7f58f7dd8b-5plg9`) | **有序且可预测** (`-0`, `-1`, `-2`) |
| **网络标识** | 不稳定 (Pod IP 随重建变化) | **稳定** (通过 Headless Service 提供可预测的 DNS 名称) |
| **存储** | 可共享一个 PVC，或使用临时存储 | **每个 Pod 都有自己独立的、持久的 PVC** |
| **部署/扩容** | 并行、无序 | **严格有序** (按 `0, 1, 2...` 的顺序创建) |
| **缩容/删除** | 无序 | **严格有序** (按 `N-1, N-2, ...0` 的逆序删除) |
| **更新策略** | 滚动更新 (RollingUpdate) | 滚动更新 (RollingUpdate, 但同样是有序的) / OnDelete |

> **一句话总结**：Deployment 管理的是一群无差别的、可以被随意替换的 Pod；而 StatefulSet 管理的是一组拥有稳定、唯一身份的 Pod，每个 Pod 都有自己绑定的持久化存储。

#### **2. Pod 的稳定网络标识**

StatefulSet 的一个“超能力”是为它的每个 Pod 提供一个稳定的网络标识。这是通过与一个特殊的 **Headless Service (无头服务)** 配合实现的。

  * **什么是 Headless Service？**
    它是一个 `clusterIP` 被显式设置为 `None` 的 Service。普通的 Service 会提供一个 ClusterIP 用于负载均衡，而 Headless Service 不提供负载均衡，它的作用是为后端匹配到的每一个 Pod，在集群内部的 DNS 系统中创建一个独立的 A 记录。

  * **DNS 记录格式**：
    `<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local`

  * **为什么这很重要？**
    对于一个数据库集群，主节点需要知道所有从节点的地址来进行数据同步。从节点之间也可能需要互相通信。通过这些可预测的、稳定的 DNS 名称（例如 `mysql-0.mysql-svc`, `mysql-1.mysql-svc`），集群成员之间就可以方便地互相发现和通信，而无需关心它们随时可能变化的 Pod IP 地址。

#### **3. StatefulSet 常见场景**

任何需要满足以下一个或多个条件的应用，都应该使用 StatefulSet：

  * **稳定的、唯一的网络标识符**。
  * **稳定的、持久的存储**。
  * **有序的、平滑的部署和伸缩**。
  * **有序的、自动的滚动更新**。

**具体例子**：

  * **数据库集群**：MySQL, PostgreSQL, MongoDB, Elasticsearch
  * **消息队列集群**：Kafka, RabbitMQ, Zookeeper
  * **分布式键值存储**：Redis Cluster, etcd

-----

### **实战任务**

我们将部署一个有 2 个副本的 Nginx StatefulSet。虽然 Nginx 本身是无状态的，但用它来演示 StatefulSet 的核心特性（有序命名、独立 PVC）非常直观和简单。

#### **第 1 步: 创建 Headless Service**

StatefulSet 需要一个 Headless Service 来提供网络标识。

1.  **新建 `nginx-headless-service.yaml` 文件**:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-svc # 这个名字很重要，StatefulSet 会引用它
    spec:
      clusterIP: None # <-- 关键！这让它成为一个 Headless Service
      selector:
        app: nginx-sts # 匹配将由 StatefulSet 创建的 Pod
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    ```

#### **第 2 步: 创建 StatefulSet**

1.  **新建 `nginx-statefulset.yaml` 文件**:
    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: nginx-sts
    spec:
      serviceName: "nginx-svc" # <-- 关键！必须匹配上面的 Headless Service 的名字
      replicas: 2
      selector:
        matchLabels:
          app: nginx-sts
      template:
        metadata:
          labels:
            app: nginx-sts
        spec:
          containers:
          - name: nginx
            image: nginx:1.24
            ports:
            - containerPort: 80
            volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html

      # --- 关键部分：PVC 模板 ---
      volumeClaimTemplates:
      - metadata:
          name: www # PVC 的名字前缀
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi
    ```
    **YAML 解读 `volumeClaimTemplates`**: 这是 StatefulSet 的精髓。它是一个 PVC 的模板。当 StatefulSet 创建一个 Pod（如 `nginx-sts-0`）时，它会自动使用这个模板为该 Pod 创建一个专属的 PVC（名为 `www-nginx-sts-0`）。创建 `nginx-sts-1` 时，会自动创建 `www-nginx-sts-1`，以此类推。

#### **第 3 步: 部署并观察**

1.  **应用这两个文件**:
    `kubectl apply -f nginx-headless-service.yaml`
    `kubectl apply -f nginx-statefulset.yaml`

2.  **观察 Pod 的创建顺序**:
    `kubectl get pods --watch`
    你会清晰地看到，`nginx-sts-0` 会先进入 `Running` 状态，然后 `nginx-sts-1` 才会开始创建。这是有序部署的体现。

3.  **观察 Pod 和 PVC 的命名**:

      * 查看 Pods: `kubectl get pods`

        ```
        NAME          READY   STATUS    RESTARTS   AGE
        nginx-sts-0   1/1     Running   0          2m
        nginx-sts-1   1/1     Running   0          90s
        ```

        名称是有序的，从 0 开始。

      * 查看 PVCs: `kubectl get pvc`

        ```
        NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
        www-nginx-sts-0   Bound    pvc-f7a6b1e0-c8e4-4a0b-8d7b-12345abcdef   1Gi        RWO            standard       2m
        www-nginx-sts-1   Bound    pvc-0e3d1c9f-3b7c-4e8a-9f5e-67890abcdef   1Gi        RWO            standard       90s
        ```

        PVC 被自动创建，并与 Pod 的名字一一对应。

#### **第 4 步: 写入唯一数据**

为了验证每个 Pod 都挂载了自己的独立存储，我们分别向它们的首页写入各自的 Pod Name。

1.  向 `nginx-sts-0` 写入数据:
    `kubectl exec nginx-sts-0 -- sh -c 'echo "I am Pod nginx-sts-0" > /usr/share/nginx/html/index.html'`

2.  向 `nginx-sts-1` 写入数据:
    `kubectl exec nginx-sts-1 -- sh -c 'echo "I am Pod nginx-sts-1" > /usr/share/nginx/html/index.html'`

#### **第 5 步: 测试身份和存储的稳定性**

现在，我们模拟 `nginx-sts-0` 这个 Pod 发生故障，删除它，看看会发生什么。

1.  **删除 Pod**:
    `kubectl delete pod nginx-sts-0`

2.  **观察重建过程**:
    `kubectl get pods --watch`
    你会看到，旧的 `nginx-sts-0` 被 `Terminating` 后，Kubernetes **并没有**创建一个随机名字的新 Pod，而是重新创建了一个**名字完全相同**的 Pod：`nginx-sts-0`！这就是**稳定的身份**。

3.  **验证数据**：
    等待新的 `nginx-sts-0` 进入 `Running` 状态后，我们来读取它的首页内容。
    `kubectl exec nginx-sts-0 -- cat /usr/share/nginx/html/index.html`

    你会看到输出：

    ```
    I am Pod nginx-sts-0
    ```

    数据完美无缺！这证明了新的 `nginx-sts-0` Pod 被创建后，自动地重新挂载了它之前专属的 PVC `www-nginx-sts-0`，从而恢复了它的状态。

-----

### **清理环境**

```bash
# 删除 StatefulSet 和 Service
kubectl delete statefulset nginx-sts
kubectl delete service nginx-svc

# ⚠️ 注意：StatefulSet 删除后，为了保护数据，它创建的 PVC 不会 被自动删除！
# 你需要手动删除它们。
kubectl delete pvc www-nginx-sts-0 www-nginx-sts-1
```

-----

### **本日总结**

今天你掌握了 Kubernetes 中用于管理有状态应用的核心武器——**StatefulSet**：

1.  **理解了 StatefulSet 与 Deployment 的核心区别**：关键在于“身份”，StatefulSet 为每个 Pod 提供了稳定、唯一的身份。
2.  **掌握了 StatefulSet 的三大特性**：稳定的 Pod 命名、稳定的网络标识（通过 Headless Service）、以及稳定的独立存储（通过 `volumeClaimTemplates`）。
3.  **亲手验证了 StatefulSet Pod 在被删除重建后，能够保持其身份并重新挂载原有的存储卷**，从而保证了服务的状态连续性。

至此，你已经学习了 Kubernetes 中最主要的两种工作负载控制器：用于无状态应用的 Deployment 和用于有状态应用的 StatefulSet。你已经具备了部署绝大多数类型应用的能力！