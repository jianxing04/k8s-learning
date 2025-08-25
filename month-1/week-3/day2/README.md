好的，欢迎来到学习计划的第 16 天！今天我们将深入探讨 Kubernetes 的流量管理核心——Ingress Controller。我会一步步带你掌握从理论到实践的全部内容，确保你完成今天的学习目标。

-----

### 🎯 学习内容：理论知识篇

在我们开始动手之前，必须先理解几个核心概念。

#### 1\. Ingress 与 Ingress Controller 的关系

想象一下，你住在一个大型公寓楼里，楼里有很多住户（**Services**）。公寓楼的大门就是集群的入口。

  * **Ingress**: 就像是门卫处的一本**规则登记簿**。它上面写着：“所有找 ‘张三’（`service-a`）的访客，请走左边的通道（`/path-a`）”；“所有找 ‘李四’（`service-b`）的访客，请走右边的通道（`/path-b`）”。Ingress 本身只是一个 Kubernetes 的 API 对象，它只定义了规则，但它自己**不能**执行这些规则。

  * **Ingress Controller**: 这就是那位**真正的门卫**。他会时刻盯着这本规则登记簿（Ingress）。每当登记簿上有新的规则或规则更新时，他就会立即按照新的规则来指挥交通。当外部流量到达时，门卫（Ingress Controller）会检查流量的目的地（例如 `hostname` 或 `path`），然后根据登记簿上的规则，将流量准确地引导到对应的住户（Service）那里。

**总结一下：**

| 组件 | 角色 | 实体 |
| :--- | :--- | :--- |
| **Ingress** | **规则 (What)** | 一个 `YAML` 文件，定义了 HTTP/HTTPS 路由规则。 |
| **Ingress Controller** | **执行者 (How)** | 一个正在运行的 Pod，通常是反向代理（如 Nginx、Traefik），它读取 Ingress 规则并应用它们。 |

**没有 Ingress Controller，Ingress 资源将不会起任何作用。**

#### 2\. 常见 Ingress Controller

社区中有许多 Ingress Controller 的实现，它们基于不同的代理技术，各有优劣。

  * **Nginx Ingress Controller**: 这是目前最流行、最成熟的 Ingress Controller。

      * **优点**: 基于高性能的 Nginx 反向代理，功能非常丰富，社区庞大，文档完善，稳定可靠。
      * **维护方**: Kubernetes 社区 (ingress-nginx) 和 NGINX 公司 (nginx-ingress) 维护着两个不同的版本，我们通常使用的是社区版。Minikube 默认安装的也是社区版。

  * **Traefik**: 这是一个现代化的、为云原生环境设计的反向代理和负载均衡器。

      * **优点**: 配置简单，能够自动发现服务（Service Discovery），与 Let's Encrypt 集成紧密，可以轻松实现 HTTPS 证书的自动申请和续期。对初学者非常友好。

今天我们的实战将使用 Minikube 自带的 Nginx Ingress Controller。

#### 3\. Ingress 配置多路径、多 Host 的方法

Ingress 的强大之处在于其灵活的路由规则。

**A. 多路径 (Path-based Routing)**

这是最常见的用法，即根据访问的 URL 路径将流量路由到不同的服务。这正是我们今天要实战的内容。

**示例 YAML:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: my-app.com # 可以是任意域名
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

这个规则意味着：

  * 访问 `http://my-app.com/app1` 的流量会被转发到 `app1-service`。
  * 访问 `http://my-app.com/app2` 的流量会被转发到 `app2-service`。

**B. 多 Host (Host-based Routing)**

你可以根据请求的域名（Host header）将流量路由到不同的服务。

**示例 YAML:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: app1.my-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.my-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

这个规则意味着：

  * 访问 `http://app1.my-domain.com` 的流量会被转发到 `app1-service`。
  * 访问 `http://app2.my-domain.com` 的流量会被转发到 `app2-service`。

理论学习结束，让我们进入实战！

-----

### 🎯 实战任务：动手操作篇

请打开你的终端，我们一步步来完成。

#### 第 1 步：启动 Minikube 并启用 Ingress 插件

首先，确保你的 Minikube 正在运行。

```bash
minikube start
```

接下来，执行今天的第一个命令，启用 ingress 插件。

```bash
minikube addons enable ingress
```

这个命令会在你的 Minikube 集群中创建一个名为 `ingress-nginx` 的 namespace，并部署 Nginx Ingress Controller 相关的 Pod。

**验证 Ingress Controller 是否启动成功：**

```bash
# 等待几秒钟，让 Pod 启动
kubectl get pods -n ingress-nginx
```

你应该能看到一个名为 `ingress-nginx-controller-xxxx` 的 Pod 正在运行（`Running` 状态）。

```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-vcl8b        0/1     Completed   0          68s
ingress-nginx-admission-patch-9f2m2         0/1     Completed   1          68s
ingress-nginx-controller-65b99975cd-bclj6   1/1     Running     0          68s
```

#### 第 2 步：部署两个服务：app1 和 app2

为了方便，我们将两个应用的 Deployment 和 Service 定义放在一个 YAML 文件里。请创建一个名为 `apps-deployment.yaml` 的文件，并粘贴以下内容：

```yaml
# apps-deployment.yaml

# ----------------- App1 ----------------- #
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1-container
        # 使用一个可以回显请求信息的镜像，方便我们验证
        image: gcr.io/google-samples/echoserver:1.10
        ports:
        - containerPort: 8080
        env:
        - name: "NODE_NAME"
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: "POD_NAME"
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: "POD_NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: "POD_IP"
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080

# ----------------- App2 ----------------- #
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      # 使用同一个镜像，但它会回显自己的 Pod 名称，所以我们可以区分
      - name: app2-container
        image: gcr.io/google-samples/echoserver:1.10
        ports:
        - containerPort: 8080
        env:
        - name: "NODE_NAME"
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: "POD_NAME"
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: "POD_NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: "POD_IP"
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

现在，应用这个文件来创建资源：

```bash
kubectl apply -f apps-deployment.yaml
```

**验证服务是否部署成功：**

```bash
kubectl get deployment,service
```

你应该能看到 `app1-deployment`, `app2-deployment`, `app1-service`, `app2-service` 这些资源。

#### 第 3 步：配置 Ingress

这是今天的核心任务。创建一个名为 `my-ingress.yaml` 的文件，并粘贴以下内容。这个配置将实现我们的目标：`http://minikube/app1` 路由到 `app1`，`http://minikube/app2` 路由到 `app2`。

```yaml
# my-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    # 这一行对于 Nginx Ingress Controller 很重要，它能更好地处理路径重写
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  # 我们这里不指定 host，Ingress Controller 将会为所有指向其 IP 的请求应用此规则
  # 这种方式在 minikube 环境下最简单
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

现在，应用这个 Ingress 规则：

```bash
kubectl apply -f my-ingress.yaml
```

**验证 Ingress 是否创建成功：**

```bash
kubectl get ingress
```

你会看到名为 `my-app-ingress` 的 Ingress 资源。注意 `ADDRESS` 字段可能需要一点时间才会分配上 IP 地址。

```
NAME             CLASS   HOSTS   ADDRESS        PORTS   AGE
my-app-ingress   nginx   * 192.168.49.2   80      30s
```

#### 第 4 步：验证外部访问路由

现在，万事俱备，只差验证！

**首先，获取 Minikube 的 IP 地址**，这是我们从外部访问 Ingress Controller 的入口。

```bash
MINIKUBE_IP=$(minikube ip)
echo $MINIKUBE_IP
```

**然后，使用 `curl` 命令来测试路由：**

1.  **访问 `/app1` 路径：**

    ```bash
    curl http://$MINIKUBE_IP/app1
    ```

    **预期输出**: 你会看到来自 `app1` Pod 的响应信息，其中 `POD_NAME` 字段会包含 `app1-deployment-xxxx`。

    ```
    Client Information:
        client_address=172.17.0.1
        command=GET
        real path=/
        query=
        request_version=1.1
        request_uri=http://192.168.49.2:8080/

    Server values:
        server_version=nginx: 1.13.3 - lua: 10008

    Request Information:
        ...
    Pod Information:
        NODE_NAME=minikube
        POD_NAME=app1-deployment-55d5b7c77c-abcde  <-- 看这里！
        POD_NAMESPACE=default
        POD_IP=172.17.0.7
    ```

2.  **访问 `/app2` 路径：**

    ```bash
    curl http://$MINIKUBE_IP/app2
    ```

    **预期输出**: 你会看到来自 `app2` Pod 的响应信息，其中 `POD_NAME` 字段会包含 `app2-deployment-xxxx`。

    ```
    Client Information:
        client_address=172.17.0.1
        command=GET
        real path=/
        query=
        request_version=1.1
        request_uri=http://192.168.49.2:8080/

    Server values:
        server_version=nginx: 1.13.3 - lua: 10008

    Request Information:
        ...
    Pod Information:
        NODE_NAME=minikube
        POD_NAME=app2-deployment-6856f6859-fghij  <-- 看这里！
        POD_NAMESPACE=default
        POD_IP=172.17.0.8
    ```

如果你看到了以上不同的 `POD_NAME`，那么恭喜你！你已经成功配置了 Ingress，并正确地将外部流量根据路径分发到了不同的后端服务。

-----

### 🧹 清理环境

完成实验后，按照“从哪来，到哪去”的原则，我们按相反的顺序删除所有创建的资源，保持环境整洁。

1.  **删除 Ingress 规则：**

    ```bash
    kubectl delete -f my-ingress.yaml
    ```

2.  **删除应用和和 Service：**

    ```bash
    kubectl delete -f apps-deployment.yaml
    ```

3.  **禁用 Ingress 插件 (可选，如果你后续不再使用)：**

    ```bash
    minikube addons disable ingress
    ```

4.  **停止或删除 Minikube 集群 (如果需要)：**

    ```bash
    # 停止集群
    minikube stop

    # 或者，彻底删除集群
    # minikube delete
    ```

-----

### 总结

今天，你完成了第 16 天的学习计划，收获满满：

1.  **理解了核心概念**：你明白了 Ingress 是“规则”，而 Ingress Controller 是实现规则的“执行者”。
2.  **了解了主流方案**：你知道了 Nginx Ingress 和 Traefik 是两种常见的 Ingress Controller。
3.  **掌握了配置方法**：你学会了如何通过 YAML 配置多路径和多 Host 的路由规则。
4.  **完成了实战演练**：你亲手在 Minikube 上部署了 Ingress Controller，并通过它将外部流量成功路由到了两个不同的服务。

Ingress 是将你的应用暴露给外部世界的标准方式，是 Kubernetes 中非常重要的一环。熟练掌握它，你的 K8s 技能将更上一层楼！继续努力！