太棒了，我们来到了 Day 6，一个激动人心的综合实战日！

经过前五天的学习，你已经掌握了 Kubernetes 中从 Pod 到 Ingress 的核心概念。今天，我们不做"理论学习"，而是将所有知识点像乐高积木一样拼装起来，从零开始部署一个完整的、可通过域名访问的 Web 应用。

这将是你学习成果的一次全面检验和展示。准备好了吗？让我们开始吧！

-----

### 🎯 **目标 6：将本周所学知识融会贯通**

-----

我们将把以下所有知识点串联起来：

  * **Day 1 (Pod)**: 作为我们应用运行的基本单元（在 Deployment 模板中定义）。
  * **Day 2 (Deployment)**: 负责应用的部署、副本保持和滚动更新。
  * **Day 3 (Service)**: 为应用提供一个稳定、集群内部的访问入口和负载均衡。
  * **Day 4 (Ingress)**: 将应用通过域名和路径暴露到集群外部，实现 L7 路由。

#### **选择我们的应用**

为了让结果更直观，我们不使用简单的 Nginx。我们将使用一个专门的 `http-echo` 服务。这个服务的功能很简单：它会把你发送给它的 HTTP 请求信息（包括路径、Header等）以文本形式返回给你。

我们将使用 `hashicorp/http-echo` 这个镜像，并通过启动参数让它返回一个我们指定的字符串。这样做的好处是，在滚动更新时，我们可以通过返回字符串的变化，清晰地看到版本切换的过程。

-----

### **实战任务：一步步构建完整应用栈**

#### **第 1 步：创建 Deployment**

首先，我们需要一个 Deployment 来管理我们的 `http-echo` Pod，我们先部署 `v1` 版本。

1.  **创建 `web-app-deployment.yaml` 文件**:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-echo-deployment
    spec:
      replicas: 2 # 我们运行 2 个副本以验证负载均衡
      selector:
        matchLabels:
          app: web-echo
      template:
        metadata:
          labels:
            app: web-echo
            version: v1 # 给 Pod 打上版本标签
        spec:
          containers:
          - name: app
            image: hashicorp/http-echo:0.2.3
            args: # 通过 args 传递启动参数
            - "-text=VERSION 1 - Hello from Kubernetes!"
            ports:
            - containerPort: 5678 # http-echo 默认监听 5678 端口
    ```

2.  **应用配置**：

    `kubectl apply -f web-app-deployment.yaml`

3.  **检查 Pod**：
    确保两个 Pod 都已成功运行。

    `kubectl get pods -l app=web-echo`

    ```bash
    NAME                                   READY   STATUS    RESTARTS   AGE
    web-echo-deployment-5c6f66788d-abcde   1/1     Running   0          20s
    web-echo-deployment-5c6f66788d-fghij   1/1     Running   0          20s
    ```

-----

#### **第 2 步：创建 Service**

现在，Pod 已经在运行，但它们是“孤岛”。我们需要一个 Service 来为它们提供一个统一的内部入口。

1.  **创建 `web-app-service.yaml` 文件**:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: web-echo-service
    spec:
      type: ClusterIP # Ingress 的后端通常是 ClusterIP
      selector:
        app: web-echo # **关键**: 这个选择器必须匹配 Deployment 创建的 Pod 标签
      ports:
      - name: http
        protocol: TCP
        port: 80 # Service 自己暴露的端口
        targetPort: 5678 # 转发到 Pod 的 5678 端口
    ```

2.  **应用配置**：

    `kubectl apply -f web-app-service.yaml`

3.  **检查 Service**：

    `kubectl get service web-echo-service`

    ```bash
    NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    web-echo-service   ClusterIP   10.100.200.30   <none>        80/TCP    10s
    ```

-----

#### **第 3 步：配置 Ingress**

我们的应用现在在集群内部是可访问的，接下来就是把它暴露给外部世界。

1.  **确保 Ingress Controller 已启用**：(如果 Day 4 后禁用了)

    `minikube addons enable ingress`

2.  **创建 `web-app-ingress.yaml` 文件**:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: web-app-ingress
      annotations:
        # 这个注解很重要，它告诉 Nginx Ingress Controller 如何重写路径
        # 将 /app/some/path 重写为 /some/path 发送给后端
        nginx.ingress.kubernetes.io/rewrite-target: /$2
    spec:
      rules:
      - host: "web.local"
        http:
          paths:
          # (.*) 是一个正则表达式，捕获 /app/ 后面的所有内容
          - path: /app(/|$)(.*)
            pathType: ImplementationSpecific # 使用正则表达式需要这个类型
            backend:
              service:
                name: web-echo-service # **关键**: 指向我们上一步创建的 Service
                port:
                  number: 80 # 指向 Service 的 80 端口
    ```

    > **路径重写解释**：我们希望通过 `http://web.local/app` 访问，但后端 `http-echo` 服务并不认识 `/app` 这个前缀。`rewrite-target` 注解配合 `path` 中的正则表达式，能智能地剥离掉 `/app` 前缀，将正确的请求路径转发给后端 Pod。

3.  **应用配置**：

    `kubectl apply -f web-app-ingress.yaml`

-----

#### **第 4 步：本地 DNS 解析与访问**

这部分是 Day 4 的复习。

1.  **获取 Minikube IP**：
    `minikube ip` (假设为 `192.168.49.2`)

2.  **修改本地 `hosts` 文件** (需要管理员权限)，添加一行：
    `192.168.49.2 web.local`

3.  **访问验证**：
    现在，打开终端，用 `curl` 访问我们的应用！

    `curl http://web.local/app`

    你应该会看到返回：

    ```
    VERSION 1 - Hello from Kubernetes!
    ```

    多执行几次，你可能会发现没有任何变化。这是因为 `http-echo` 的返回内容是固定的。但我们的应用栈已经**完全打通了**！

    `请求` → `hosts文件` → `minikube ip` → `Ingress Controller` → `Ingress规则` → `Service` → `某个Pod`

-----

#### **第 5 步：体验滚动更新**

这是最后一步，也是最酷的一步。我们将在**不中断服务**的情况下，将应用从 `v1` 升级到 `v2`。

1.  **准备实时监控**：
    为了直观地看到更新过程，我们要同时做两件事：

      * **终端 1**: 启动一个循环，每半秒请求一次我们的服务，实时打印返回内容。
      * **终端 2**: 监控 Pod 的变化。

    **在终端 1 中执行**:

    ```bash
    while true; do curl http://web.local/app; sleep 0.5; done
    ```

    现在，这个终端会不停地输出 "VERSION 1 - Hello from Kubernetes\!"。

    **在终端 2 中执行**:

    ```bash
    kubectl get pods -l app=web-echo --watch
    ```

    这个终端会显示当前的两个 v1 Pod，并等待变化。

2.  **执行更新**：
    修改 `web-app-deployment.yaml` 文件。我们只改两个地方：

      * `template.metadata.labels.version`: 从 `v1` 改为 `v2`。
      * `containers.args`: 里的文本改为 `"VERSION 2 - Update was successful!"`。

    修改后的文件片段：

    ```yaml
        metadata:
          labels:
            app: web-echo
            version: v2 # <-- 修改这里
        spec:
          containers:
          - name: app
            image: hashicorp/http-echo:0.2.3
            args:
            - "-text=VERSION 2 - Update was successful!" # <-- 修改这里
    ```

3.  **打开第三个终端，应用新配置**：

    `kubectl apply -f web-app-deployment.yaml`

4.  **观察！**
    现在，立刻去看终端 1 和终端 2：

      * **终端 2 (Pod 监控)**: 你会看到新的 `v2` Pod 开始 `ContainerCreating`，然后 `Running`。与此同时，旧的 `v1` Pod 开始 `Terminating`。这个过程是交替进行的。
      * **终端 1 (curl 循环)**: 这是最神奇的地方！你会看到输出开始在 "VERSION 1..." 和 "VERSION 2..." 之间**交替出现**！这表明在更新过程中，Ingress 的流量同时被转发到了新旧两个版本的 Pod 上。几秒钟后，当所有旧 Pod 都被替换掉，输出会**稳定地变为 "VERSION 2 - Update was successful\!"**。

    整个过程中，**请求没有一次中断**。这就是滚动更新的魅力！

-----

### **总结与清理**

恭喜你！你成功地将一周所学的所有知识点整合在了一起，部署并更新了一个完整的、高可用的 Web 应用。

**清理环境**：

1.  删除所有 Kubernetes 资源：

    ```bash
    kubectl delete deployment web-echo-deployment
    kubectl delete service web-echo-service
    kubectl delete ingress web-app-ingress
    ```

2.  **【重要】** 清理你的 `hosts` 文件，删除或注释掉 `web.local` 那一行。

你已经从一个 Kubernetes 新手，成长为能够独立部署和管理应用的实践者。这为你后续学习更高级的主题（如配置管理、存储、有状态应用等）打下了无比坚实的基础。为你鼓掌！