第三周的学习计划非常棒！**服务发现**和**可观测性**是 Kubernetes 中非常核心和实用的部分。我们先从服务发现开始，这部分对于理解微服务架构至关重要。

### **Service 与 DNS 核心概念**

在 Kubernetes 中，Pod 随时可能被创建或销毁，它们的 IP 地址也是动态分配的。为了让应用能够互相通信，我们需要一个稳定的网络地址。**Service** 就是解决这个问题的关键。它为一组 Pod 提供一个稳定的网络终点，并且可以对流量进行负载均衡。

我们来分别了解一下你计划中提到的几种 Service 类型：

  * **ClusterIP**: 这是默认的 Service 类型。它为 Service 分配一个集群内部的虚拟 IP 地址。这个 IP 只能在集群内部访问。它主要用于集群内部的通信，比如你的 `frontend` Pod 访问 `backend` Pod。
  * **NodePort**: 这种类型会在每个节点（Node）上打开一个特定的端口。集群外部的流量可以通过 `Node IP:NodePort` 的方式访问到这个 Service，进而到达背后的 Pod。它常用于开发和测试环境，或者当你想从集群外部直接访问 Service 时。
  * **LoadBalancer**: 这种类型通常与云服务商（如 AWS、GCP、阿里云等）集成。它会自动创建一个云上的负载均衡器，并将外部流量路由到你的 Service。这是将应用暴露给公网的最常用方式。

-----

### **Kube-DNS / CoreDNS 工作原理**

在 Kubernetes 集群内部，**CoreDNS**（或旧版中的 Kube-DNS）扮演着 DNS 服务器的角色。它的核心功能是将 Service 的名字解析为对应的 ClusterIP。

  * 每个 Service 在创建时，CoreDNS 都会为其生成一条 DNS 记录。
  * Pod 内部的 `/etc/resolv.conf` 文件会被 Kubelet 自动配置，将 CoreDNS 的 Service IP 地址作为 DNS 服务器。
  * 当一个 Pod 尝试通过 Service 名称（例如 `backend.default.svc.cluster.local`）访问另一个 Service 时，它会向 CoreDNS 发送 DNS 查询请求。
  * CoreDNS 接收到请求后，会返回该 Service 对应的 ClusterIP。
  * Pod 拿到 ClusterIP 后，就可以向这个 IP 发送网络请求，Service 会将请求负载均衡到其背后的 Pod 上。

这就是为什么你的 `frontend` Pod 能够通过 `backend` 这个名字找到 `backend` Service 的原因。

-----

### **实战任务：服务发现**

现在，我们来完成你计划中的实战任务。这个任务非常经典，能够让你直观地感受服务发现的整个流程。

#### **第一步：创建 `backend` Deployment 和 Service**

首先，我们需要一个简单的 `backend` 应用。我们使用一个简单的 NGINX 镜像作为示例，它能响应 HTTP 请求。

1.  **创建 `backend` Deployment**
    创建一个名为 `backend-deployment.yaml` 的文件，内容如下：
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          labels:
            app: backend
        spec:
          containers:
          - name: backend-container
            image: nginx:latest
            ports:
            - containerPort: 80
    ```
    运行命令来创建它：
    ```bash
    kubectl apply -f backend-deployment.yaml
    ```
2.  **创建 `backend` Service**
    现在，为 `backend` Deployment 创建一个 ClusterIP Service。创建一个名为 `backend-service.yaml` 的文件：
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend-service
    spec:
      selector:
        app: backend
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP
    ```
    请注意 `selector: app: backend`，这是 Service 如何找到它背后的 Pod 的关键。
    运行命令来创建它：
    ```bash
    kubectl apply -f backend-service.yaml
    ```

#### **第二步：创建 `frontend` Deployment**

现在，我们创建 `frontend` Deployment。为了方便演示，我们同样使用一个 `nginx` 镜像，然后进入其 Pod 内部进行测试。

1.  **创建 `frontend` Deployment**
    创建一个名为 `frontend-deployment.yaml` 的文件：
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: frontend
      template:
        metadata:
          labels:
            app: frontend
        spec:
          containers:
          - name: frontend-container
            image: nginx:latest
    ```
    运行命令来创建它：
    ```bash
    kubectl apply -f frontend-deployment.yaml
    ```

#### **第三步：验证服务发现**

现在，让我们进入 `frontend` Pod，验证它是否能够通过 DNS 名称访问 `backend` Service。

1.  **获取 `frontend` Pod 名称**
    运行以下命令来找到你的 `frontend` Pod 的名称：

    ```bash
    kubectl get pods -l app=frontend
    ```

    你会看到类似 `frontend-deployment-xxxxxx-yyyyy` 这样的名称。

2.  **进入 Pod 内部并测试**
    使用 `kubectl exec` 命令进入 Pod 的 shell：

    ```bash
    kubectl exec -it <frontend-pod-name> -- /bin/bash
    ```

    进入 Pod 后，首先确保 `curl` 或 `wget` 命令可用。然后执行 DNS 查询和网络请求：

    ```bash
    # 使用 nslookup 查询 backend Service 的 ClusterIP
    nslookup backend-service

    # 尝试访问 backend Service
    curl backend-service

    # 或者使用完整的 FQDN (Fully Qualified Domain Name)
    curl backend-service.default.svc.cluster.local
    ```

    `nslookup` 命令的输出会告诉你 `backend-service` 这个域名解析到了哪个 IP 地址，这个地址就是 `backend-service` 的 ClusterIP。
    `curl` 命令应该会成功获取到 NGINX 的默认欢迎页面 HTML。

如果你成功完成了以上步骤，那么恭喜你，你已经亲手验证了 Kubernetes 服务发现的整个流程。

如果遇到任何问题，随时告诉我！我们可以一起 debug，继续深入探索。

太棒了，清理实验环境是学习 Kubernetes 的一个重要习惯。这不仅能避免资源浪费，还能让你养成良好的实践。

我们可以通过删除之前创建的 YAML 文件或者直接删除 Deployment 和 Service 来清理环境。我将提供两种最常用的方法：

### **方法一：通过文件名清理**

这是最推荐的方式。如果你手边有之前创建的 YAML 文件，只需要将 `apply` 换成 `delete` 命令即可。Kubernetes 会根据 YAML 文件中的定义，找到对应的资源并将其删除。

```bash
# 删除 backend 的 Deployment 和 Service
kubectl delete -f backend-deployment.yaml
kubectl delete -f backend-service.yaml

# 删除 frontend 的 Deployment
kubectl delete -f frontend-deployment.yaml
```

-----

### **方法二：通过资源类型和名称清理**

如果你不小心删除了 YAML 文件，或者想快速删除某个特定的资源，可以通过指定资源类型和名称来删除。

1.  **删除 Deployment**
    ```bash
    kubectl delete deployment frontend-deployment
    kubectl delete deployment backend-deployment
    ```
2.  **删除 Service**
    ```bash
    kubectl delete service backend-service
    ```

-----

### **如何检查是否清理干净**

执行完删除命令后，你可以使用 `get` 命令来确认所有资源都已经被移除。

```bash
# 检查 Deployment 是否存在
kubectl get deployments

# 检查 Service 是否存在
kubectl get services

# 检查 Pod 是否存在
kubectl get pods
```

当这些命令的输出显示 "No resources found" 时，就说明你的实验环境已经清理干净了。

养成这个好习惯，可以让你更高效地管理你的 Kubernetes 学习环境。

-----

现在，我们完成了服务发现的基础学习。你准备好进入下一个主题：**可观测性**了吗？