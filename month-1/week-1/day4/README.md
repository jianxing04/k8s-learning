太棒了，我们进入了 Day 4 的学习！

昨天我们用 NodePort 实现了从集群外部访问服务，但你也一定感觉到了它的不便之处：端口号又高又难记，而且只能通过 IP 地址访问。在生产环境中，我们希望用 `https://www.myapp.com` 这样的标准域名和端口来访问服务。

今天，我们就来学习实现这一目标的“官方指定方案”—— **Ingress**。

-----

### 🎯 **目标 4：理解 Ingress 的路由机制，学会暴露服务**

-----

### **学习内容**

#### **1. Ingress 的作用 vs Service**

为了理解 Ingress，让我们用一个商场的比喻：

  * **Pod**：商场里具体的一家家店铺（比如优衣库、星巴克）。
  * **Service**：每家店铺的独立入口或后门。`ClusterIP` 就像店铺的内部电话，只有商场内部人员（其他 Pod）能用。`NodePort` 就像给每家店开了一个偏僻的、带特殊编号的后门（`节点IP:3xxxx`），虽然能进，但很不方便。
  * **Ingress**：**商场的正门和总服务台**。顾客（用户）只需要知道商场的地址（`www.mall.com`），到达正门后，服务台（Ingress）会根据你的需求（“我要去优衣库”、“我要去星巴克”），告诉你具体该怎么走。

**核心区别：**

| 特性     | Service (NodePort/LoadBalancer)                                | Ingress                                                                              |
| :------- | :------------------------------------------------------------- | :----------------------------------------------------------------------------------- |
| **层级** | **网络第 4 层 (L4)** - TCP/UDP                                 | **网络第 7 层 (L7)** - HTTP/HTTPS                                                    |
| **功能** | 基于 IP 和端口进行流量转发和负载均衡                           | 基于 **域名 (Hostname)** 和 **路径 (Path)** 进行智能路由，可以实现更复杂的转发规则 |
| **角色** | **暴露单个服务**。一个 NodePort 对应一个 Service。                 | **聚合多个服务**。一个 Ingress 可以根据不同的域名或路径，将流量分发到不同的 Service。    |

> **小结**：Service 负责将流量从 **Ingress** 导向正确的 **Pod** 组，而 Ingress 负责将 **集群外部的 HTTP/S 流量** 导向正确的 **Service**。它们是合作关系，不是替代关系。

#### **2. Ingress Controller (入口控制器)**

这是理解 Ingress 工作原理的**最关键概念**。

你用 YAML 创建的 `Ingress` 资源，它本身**什么也不做**。它只是一份“路由规则”的配置清单，就像是写在纸上的商场楼层指南。

真正让这份指南生效的，是 **Ingress Controller**。

  * **Ingress Controller** 是一个在集群中运行的 Pod，它本质上是一个反向代理服务器（比如 Nginx, Traefik, HAProxy）。
  * 它的工作就是持续监听集群中所有 Ingress 资源的创建和变化。
  * 一旦发现新的 Ingress 规则，它就会动态地更新自己的代理配置，然后根据这些规则来转发流量。

**工作流程：**
用户请求 → Ingress Controller (Pod) → 读取 Ingress 规则 → 转发到对应的 Service → Service 转发到后端的 Pod

所以，要使用 Ingress，**集群里必须先安装并运行一个 Ingress Controller**。不同的 Controller 有不同的功能和配置方式，我们今天学习最流行的 **Nginx Ingress Controller**。

#### **3. 基于路径的路由 (Path-based Routing)**

这是 Ingress 最常用的功能。你可以定义规则，将同一个域名下的不同 URL 路径转发到不同的服务。

例如，你可以配置 `myapp.com` 这个域名：

  * 访问 `myapp.com/web` 的流量 -\> 转发到 `frontend-service`
  * 访问 `myapp.com/api/v1` 的流量 -\> 转发到 `backend-service`
  * 访问 `myapp.com/admin` 的流量 -\> 转发到 `dashboard-service`

这样，你只需要一个公网 IP 和一个域名，就可以为整个应用系统提供统一的入口，非常强大和整洁。

-----

### **实战任务**

理论已经清晰，让我们把昨天的 Nginx 服务通过域名和路径暴露出来！

**准备工作**：我们需要一个 Deployment 和一个 ClusterIP 类型的 Service 作为 Ingress 的后端。

1.  **创建 `nginx-deployment.yaml`** (如果已删除，请重新创建):

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx-deployment
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
            image: nginx:1.24
            ports:
            - containerPort: 80
    ```

2.  **创建 `nginx-clusterip-service.yaml`** (注意，这次我们用 ClusterIP 就够了):

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx-service # 为了清晰，改个名
    spec:
      type: ClusterIP
      selector:
        app: my-nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

3.  **应用它们**：

    ```bash
    kubectl apply -f nginx-deployment.yaml
    kubectl apply -f nginx-clusterip-service.yaml
    ```

-----

#### **1. 安装 Nginx Ingress Controller**

Minikube 将安装过程简化成了一个插件命令。

1.  **启用 ingress 插件**：

    `minikube addons enable ingress`

    这个命令会在你的 Minikube 集群中自动部署 Nginx Ingress Controller 相关的所有资源。

2.  **验证安装**：
    Ingress Controller 会被安装在一个名为 `ingress-nginx` 的命名空间（namespace）下。执行以下命令查看它的 Pod 是否正常运行：

    `kubectl get pods -n ingress-nginx`

    你将看到类似下面的输出，确保它的状态是 `Running`：

    ```bash
    NAME                                        READY   STATUS      RESTARTS   AGE
    ingress-nginx-admission-create-j75l9        0/1     Completed   0          60s
    ingress-nginx-admission-patch-t9kdv         0/1     Completed   0          60s
    ingress-nginx-controller-59b45fb494-b7z2h    1/1     Running     0          60s
    ```

    很好，我们的“商场服务台”已经就位了！

-----

#### **2. 创建一个 Ingress，把 `/nginx` 路径转发到 `nginx-service`**

现在我们要创建那份“路由规则”。

1.  **创建 `nginx-ingress.yaml` 文件**：

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-app-ingress
      annotations:
        # 这个注解很重要，它告诉 Nginx Ingress Controller 如何重写路径
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: "myapp.local" # 我们要匹配的域名
        http:
          paths:
          - path: /nginx # 我们要匹配的路径
            pathType: Prefix # 路径匹配类型：前缀匹配
            backend:
              service:
                name: my-nginx-service # 要转发到的 Service 名称
                port:
                  number: 80 # Service 暴露的端口
    ```

    **注解解读 `rewrite-target: /`**:
    当用户访问 `myapp.local/nginx` 时，Ingress Controller 会匹配到这个规则。在将请求转发给后端的 `my-nginx-service` 之前，`rewrite-target: /` 会**自动去掉**请求路径中的 `/nginx` 前缀。所以后端 Nginx Pod 实际收到的请求路径是 `/`（根路径），这才能正确地访问到它的首页。如果没有这个设置，Pod 会收到对 `/nginx` 路径的请求，很可能会返回 404 Not Found。

2.  **应用 Ingress 规则**：

    `kubectl apply -f nginx-ingress.yaml`

3.  **查看 Ingress**：

    `kubectl get ingress`

    ```bash
    NAME             CLASS    HOSTS         ADDRESS          PORTS   AGE
    my-app-ingress   nginx    myapp.local   192.168.49.2     80      10s
    ```

    `ADDRESS` 列显示的正是你的 Minikube IP。

-----

#### **3. 在本地 hosts 文件配置 `myapp.local` 指向 `minikube ip`**

`myapp.local` 不是一个真实的互联网域名，你的电脑和浏览器不知道它指向哪里。我们需要手动告诉你的操作系统：“如果你看到 `myapp.local` 这个域名，请访问 `minikube ip` 这个地址”。

1.  **获取 Minikube IP**：

    `minikube ip`
    （假设输出是 `192.168.49.2`）

2.  **修改 hosts 文件**（需要管理员权限）：

      * **macOS / Linux**: `sudo nano /etc/hosts`
      * **Windows**: 以管理员身份打开记事本，然后打开 `C:\Windows\System32\drivers\etc\hosts` 文件。

    在文件末尾添加一行，将 `minikube ip` 和你的域名对应起来：

    ```
    192.168.49.2  myapp.local
    ```

    保存并退出。

3.  **验证配置**：
    打开终端，`ping` 一下你的新域名：
    `ping myapp.local`
    如果它能成功 ping 通并显示 Minikube 的 IP 地址，说明配置成功！

-----

#### **4. 用 `http://myapp.local/nginx` 访问**

激动人心的时刻到了！

打开你的浏览器，访问：

**`http://myapp.local/nginx`**

或者在命令行使用 `curl`:

`curl http://myapp.local/nginx`

你应该能立即看到 **Welcome to nginx\!** 的页面。

**让我们回顾一下完整的请求流程**：

1.  你在浏览器输入 `http://myapp.local/nginx`。
2.  你的电脑通过 `hosts` 文件，将 `myapp.local` 解析为 Minikube 的 IP (`192.168.49.2`)。
3.  请求被发送到 Minikube 节点的 80 端口，被 Ingress Controller 捕获。
4.  Ingress Controller 查看请求的 `Host` 是 `myapp.local`，`path` 是 `/nginx`，匹配到了我们定义的 Ingress 规则。
5.  根据 `rewrite-target` 注解，它将请求路径从 `/nginx` 修改为 `/`。
6.  请求被转发到名为 `my-nginx-service` 的 ClusterIP Service。
7.  Service 将请求负载均衡到后端的某个 Nginx Pod。
8.  Nginx Pod 响应欢迎页面，数据原路返回给你的浏览器。

-----

### **清理环境**

1.  **删除本次创建的 K8s 资源**：

    ```bash
    kubectl delete ingress my-app-ingress
    kubectl delete service my-nginx-service
    kubectl delete deployment my-nginx-deployment
    ```

2.  **禁用 ingress 插件** (可选，也可以留着下次用):
    `minikube addons disable ingress`

3.  **【重要】清理 hosts 文件**：
    再次以管理员身份编辑你的 `hosts` 文件，**删除或注释掉**我们添加的那一行，以免影响你未来的网络访问。

    ```
    # 192.168.49.2  myapp.local
    ```

-----

### **本日总结**

祝贺你！今天你掌握了 Kubernetes 中暴露服务的“终极形态” Ingress：

1.  **理解了 Ingress 作为集群的流量入口**，以及它和 Service 在 L7 和 L4 上的根本区别。
2.  **明白了 Ingress 规则 (YAML) 和 Ingress Controller (实际的代理软件) 的主从关系**。
3.  **亲手实践了基于域名和路径的路由**，将外部流量精确地导向了内部服务。

从 Day 1 的一个孤立的 Pod，到 Day 4 可以通过域名访问的高可用应用，你已经搭建起了一个微型但完整的 Kubernetes 应用架构。你已经掌握了 Kubernetes 最核心的几个概念，非常了不起！