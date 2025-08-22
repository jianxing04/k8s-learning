欢迎来到 Day 9！

在 Day 8，我们学习了如何通过 ConfigMap 和 Secret 将**配置数据**注入到 Pod 中，其中一种方式就是通过 Volume 文件挂载。这为你今天深入理解 Kubernetes 的存储核心——**Volume**——打下了基础。

容器生来是**无状态 (Stateless)** 的，它们的文件系统是临时的。当容器崩溃或 Pod 被销毁时，其中所有的数据都会丢失。但现实中的应用（如数据库、文件服务器、用户上传内容等）都需要**持久化 (Persistence)** 的数据。今天，我们就来解决这个至关重要的问题。

-----

### 🎯 **目标 9：理解存储卷的概念，掌握 Pod 的持久化**

-----

### **学习内容**

#### **1. Volume：Pod 生命周期内共享数据**

首先，要理解 Volume 的核心概念：
一个 Kubernetes Volume 是一个**独立于容器文件系统**的数据目录，它可以被 Pod 中的一个或多个容器挂载和访问。

**核心特性**：

  * **数据共享**：同一个 Volume 可以被一个 Pod 内的多个容器同时挂载，是实现容器间文件共享的主要方式。
  * **生命周期**：Volume 的生命周期与**它所在的 Pod**绑定。只要 Pod 存在，Volume 就会存在，即使里面的某个容器崩溃重启，Volume 中的数据也不会丢失。**但是，如果 Pod 被删除，Volume（以及其中数据）通常也会被销毁**（除非是持久化类型的 Volume）。

> **比喻**：Volume 就像是挂载在 Pod 上的一个“共享U盘”。Pod 里的所有容器都可以读写这个U盘。只要 Pod 没被销毁，这个U盘就一直插在上面，数据就不会丢。

#### **2. `emptyDir` 与 `hostPath` 卷**

Kubernetes 支持多种类型的 Volume，我们先看两种最基础的：

  * **`emptyDir`**

      * **是什么**：一个临时的空目录。当 Pod 被分配到某个节点上时，Kubernetes 会为它创建一个 `emptyDir` 卷，Pod 内的容器可以读写这个目录。
      * **生命周期**：完全等同于 Pod 的生命周期。**当 Pod 因任何原因被删除时，`emptyDir` 中的数据将永久丢失**。
      * **典型用途**：
        1.  **容器间共享文件（我们今天的实战）**：一个容器向 `emptyDir` 写数据，另一个容器从中读取。
        2.  **临时暂存空间**：作为应用运行时的临时缓存或排序空间。

  * **`hostPath`**

      * **是什么**：将\*\*宿主节点（Node）\*\*上的一个文件或目录，直接挂载到 Pod 的文件系统中。
      * **生命周期**：数据存储在节点上，如果 Pod 被删除并重新创建（即使在同一个节点上），数据依然存在。
      * **⚠️ 警告**：这是一个强大但危险的功能，**不推荐用于绝大多数应用**。因为它破坏了 Pod 与节点的解耦，你的应用被“钉死”在了某个特定的节点上。如果 Pod 被调度到其他节点，数据就访问不到了。
      * **典型用途**：用于需要读取或修改节点信息的系统级应用，比如监控 Agent（读取节点日志）、网络插件等。

#### **3. PersistentVolume (PV) 与 PersistentVolumeClaim (PVC)**

`emptyDir` 和 `hostPath` 都无法满足应用数据在 Pod 销毁后依然存在的“持久化”需求。为此，Kubernetes 设计了一套强大的抽象机制：PV 和 PVC。

这套机制将“存储资源的管理”和“存储资源的使用”这两个角色完美地分离开来。

  * **PersistentVolume (PV) - 持久卷**

      * **角色**：**“存储资源”**，由**集群管理员 (Administrator)** 创建和管理。
      * **是什么**：代表集群中一块已经存在的网络存储（如 NFS、Ceph、云服务商的磁盘等）。它和具体的 Pod 没有关系，是集群级别的资源。PV 拥有独立于任何 Pod 的生命周期。
      * **比喻**：PV 就像是IT部门提前采购好的一批\*\*“移动硬盘”\*\*，放在仓库里，每个硬盘都有自己的容量（10GB）、速度（SSD）、接口类型（读写模式）等规格。

  * **PersistentVolumeClaim (PVC) - 持久卷声明**

      * **角色**：**“存储请求”**，由**应用开发者 (Developer)** 创建和使用。
      * **是什么**：用户对存储资源的一次申请。开发者在 PVC 中声明自己需要多大的空间、什么样的读写权限等，而**不关心**这些存储具体来自哪里。
      * **比喻**：PVC 就像是你（开发者）填写的一张\*\*“移动硬盘领用申请单”\*\*：“我需要一个至少 1GB、可以单人读写的硬盘。”

**工作流程 (动态供给 - Dynamic Provisioning)**:
在现代 Kubernetes 集群中（包括 Minikube），通常会配置一个 `StorageClass` 来实现**动态供给**。

1.  开发者创建一个 PVC (提交申请单)。
2.  Kubernetes 看到这个 PVC，会通过 `StorageClass` 的配置，**自动地**在底层存储系统（如云硬盘）中创建一个符合要求的 PV (自动从仓库拿一个新硬盘)。
3.  这个新创建的 PV 会自动与开发者的 PVC **绑定 (Bound)**。
4.  开发者在自己的 Pod 中，只需要引用 PVC 的名字，就可以像使用普通 Volume 一样，将这块持久化存储挂载到容器中。

> **核心思想**：开发者只需要关心“我需要什么样的存储”(PVC)，而无需关心“存储到底从哪里来”(PV)。这大大简化了应用开发和部署的复杂度。

-----

### **实战任务**

#### **任务 1：使用 `emptyDir` 卷共享文件**

**场景**：创建一个 Pod，包含两个容器。一个容器（`writer`）每秒向共享目录写入当前时间，另一个容器（`reader`）每秒从该目录读取并显示时间。

1.  **创建 `emptydir-pod.yaml` 文件**：

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: emptydir-demo-pod
    spec:
      # 1. 在 Pod 层面定义一个 emptyDir 类型的 Volume
      volumes:
      - name: shared-data
        emptyDir: {}

      containers:
      # 2. 第一个容器：写入者
      - name: writer
        image: busybox:1.35
        command: ["/bin/sh", "-c", "while true; do echo $(date) > /data/time.txt; sleep 1; done"]
        # 3. 将 Volume 挂载到容器的 /data 目录
        volumeMounts:
        - name: shared-data
          mountPath: /data

      # 4. 第二个容器：读取者
      - name: reader
        image: busybox:1.35
        command: ["/bin/sh", "-c", "while true; do cat /data/time.txt; sleep 1; done"]
        # 5. 将同一个 Volume 也挂载到自己的 /data 目录
        volumeMounts:
        - name: shared-data
          mountPath: /data
    ```

2.  **部署并验证**：
    `kubectl apply -f emptydir-pod.yaml`
    实时查看 `reader` 容器的日志：
    `kubectl logs emptydir-demo-pod -c reader --follow`

    你会看到，屏幕上会实时刷新由 `writer` 容器写入的时间戳，证明两个容器通过 `emptyDir` 成功共享了文件。

-----

#### **任务 2 & 3：使用 PVC 持久化 Nginx 静态页面**

**场景**：我们将创建一个 PVC，把它挂载到 Nginx Pod 的网站根目录。然后我们在里面写入一个自定义首页。接着删除 Pod 再重建，验证首页文件是否依然存在。

1.  **第 A 步: 创建 PVC (提交“硬盘申请单”)**
    新建 `my-pvc.yaml` 文件：

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: my-nginx-pvc
    spec:
      accessModes:
        - ReadWriteOnce # 申请的读写模式：单节点读写
      resources:
        requests:
          storage: 1Gi # 申请的存储空间大小：1 GiB
      # storageClassName: standard # 在 Minikube 中可以不写，它会使用默认的
    ```

    应用它： `kubectl apply -f my-pvc.yaml`
    查看状态：`kubectl get pvc`，你会看到它的状态很快从 `Pending` 变为 `Bound`。

2.  **第 B 步: 创建使用 PVC 的 Pod**
    新建 `nginx-pvc-pod.yaml`：

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pvc-pod
    spec:
      volumes:
        # 定义一个 Volume，它的类型是 PVC
        - name: nginx-storage
          persistentVolumeClaim:
            claimName: my-nginx-pvc # 引用我们上一步创建的 PVC
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        # 将这个 Volume 挂载到 Nginx 的网站根目录
        volumeMounts:
        - name: nginx-storage
          mountPath: /usr/share/nginx/html
    ```

    应用它： `kubectl apply -f nginx-pvc-pod.yaml`

3.  **第 C 步: 向持久卷写入数据**
    `kubectl exec -it nginx-pvc-pod -- /bin/bash`
    进入 Pod 的 shell 后，写入一个自定义的 `index.html`：
    `echo '<h1>Hello from Persistent Volume!</h1>' > /usr/share/nginx/html/index.html`
    输入 `exit` 退出。
    可以通过端口转发验证一下页面内容：
    `kubectl port-forward nginx-pvc-pod 8080:80`
    然后在另一个终端 `curl localhost:8080`，你应该能看到我们写入的 HTML。

4.  **第 D 步: 见证奇迹 - 删除并重建 Pod**
    删除 Pod：
    `kubectl delete pod nginx-pvc-pod`
    确认 Pod 已被删除。此时，**关键在于**，我们的 PVC 依然存在： `kubectl get pvc`，你会看到 `my-nginx-pvc` 还在。

    现在，**重新创建完全相同的 Pod**：
    `kubectl apply -f nginx-pvc-pod.yaml`

5.  **第 E 步: 验证数据是否还在**
    等待新的 Pod 变为 `Running` 状态。
    再次设置端口转发：
    `kubectl port-forward nginx-pvc-pod 8080:80`
    再次 `curl localhost:8080`...

    你将再次看到 **"Hello from Persistent Volume\!"**。数据完美地保留了下来！这证明了 PV/PVC 机制成功地将数据的生命周期与 Pod 的生命周期分离开来。

-----

### **清理环境**

```bash
kubectl delete pod emptydir-demo-pod
kubectl delete pod nginx-pvc-pod
kubectl delete pvc my-nginx-pvc # 删除 PVC 会触发动态供给的 PV 被自动回收删除
```

-----

### **本日总结**

今天，你掌握了 Kubernetes 中至关重要的存储概念：

1.  **理解了 Volume 的基本作用**：为 Pod 内的容器提供共享和临时存储。
2.  **掌握了 `emptyDir` 的用法**：适用于 Pod 内的临时数据交换。
3.  **彻底理解了 PV 和 PVC 的抽象机制**：这是 Kubernetes 实现持久化存储的核心，通过将“存储实现”与“存储消费”解耦，提供了极大的灵活性和可移植性。
4.  **亲手验证了使用 PVC 的数据可以跨越 Pod 的生死而持久存在**。

掌握了持久化存储，你就解锁了在 Kubernetes 上运行**有状态应用 (Stateful Applications)**（如数据库、消息队列等）的能力。明天，我们将学习专门为这类应用设计的控制器：**StatefulSet**。