太棒了！欢迎来到第 21 天，这是我们这一周学习的“毕业日”！今天没有新的知识点，但有更重要的任务：将过去几天学到的所有核心概念——`StatefulSet`, `PVC`, `ConfigMap`, `Secret`, `Ingress`, `NetworkPolicy`, `Probes`, `Metrics`——全部融合在一起，像搭乐高一样，构建出一个相对完整、健壮且安全的微服务应用。

这是对你学习成果的一次全面检验和升华。准备好成为一名真正的 Kubernetes 应用架构师了吗？我们开始吧！

-----

### 🎯 综合实战任务

我们将部署一个经典的三层应用：一个简单的访客留言板。

  * **Frontend (Web 层)**: 一个 Nginx 服务器，接收用户请求。
  * **Backend (API 层)**: 一个 Go 语言编写的 API 服务，负责处理业务逻辑（读/写留言）。
  * **Database (数据层)**: 一个 Redis 数据库，用于存储留言数据。

我们将一步步为这个应用添加上安全、持久化和可观测性的“铠甲”。

#### 第 0 步：环境准备

这个综合实验需要用到前几天的多个组件，特别是 `ingress` 和 `calico` (用于 NetworkPolicy)。

请确保你使用一个“干净”的 Minikube 环境，并用以下命令启动它：

```bash
# 确保旧的已删除
minikube delete
# 启动并安装 Calico CNI
minikube start --network-plugin=cni --cni=calico

# 启用我们需要的插件
minikube addons enable ingress
minikube addons enable metrics-server
```

**请耐心等待所有插件和组件启动完成。**

#### 第 1 步：部署 Database 层 (StatefulSet + PVC + Secret)

我们从最底层的数据库开始。数据是最核心的资产，所以我们必须保证它的安全和持久化。

1.  **创建 Redis 密码 (Secret)**
    我们绝不能将密码明文写在配置里。使用 `Secret` 是最佳实践。

    ```bash
    # 创建一个值为 "supersecret" 的密码
    kubectl create secret generic redis-secret --from-literal=password=supersecret
    ```

2.  **创建 Database 的 Manifest 文件**
    创建一个 `database-layer.yaml` 文件。这个文件将包含用于 Redis 的 `StatefulSet` 和 `Service`。

    ```yaml
    # database-layer.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-service
      labels:
        app: my-three-tier-app
        tier: database
    spec:
      ports:
      - port: 6379
      clusterIP: None # Headless Service，用于 StatefulSet 的稳定网络标识
      selector:
        app: my-three-tier-app
        tier: database
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: redis-statefulset
    spec:
      serviceName: "redis-service"
      replicas: 1
      selector:
        matchLabels:
          app: my-three-tier-app
          tier: database
      template:
        metadata:
          labels:
            app: my-three-tier-app
            tier: database # 关键标签，用于 NetworkPolicy
        spec:
          containers:
          - name: redis
            image: redis:6.2-alpine
            command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"]
            env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
            ports:
            - containerPort: 6379
            volumeMounts:
            - name: redis-data
              mountPath: /data
            # 为 Redis 添加存活探针
            livenessProbe:
              tcpSocket:
                port: 6379
              initialDelaySeconds: 15
              periodSeconds: 20
      # 使用 volumeClaimTemplates 动态创建 PVC
      volumeClaimTemplates:
      - metadata:
          name: redis-data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi
    ```

    应用它：

    ```bash
    kubectl apply -f database-layer.yaml
    ```

    **检查**：`kubectl get statefulset`, `kubectl get pvc`。你会看到一个 `redis-statefulset-0` Pod 和一个绑定的 PVC。

#### 第 2 步：部署 Backend 层 (Deployment + ConfigMap + Probes)

Backend 是连接 Frontend 和 Database 的桥梁。

1.  **创建 Backend 的 Manifest 文件**
    创建一个 `backend-layer.yaml` 文件。我们将使用一个现成的 Go API 镜像 `dockersamples/guester-api-go:0.1`。

    ```yaml
    # backend-layer.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend-service
      labels:
        app: my-three-tier-app
        tier: backend
    spec:
      ports:
      - port: 80
        targetPort: 8080
      selector:
        app: my-three-tier-app
        tier: backend
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: my-three-tier-app
          tier: backend
      template:
        metadata:
          labels:
            app: my-three-tier-app
            tier: backend # 关键标签
        spec:
          containers:
          - name: backend-api
            image: mlabouardy/guester-api-go:0.1
            ports:
            - containerPort: 8080
            env:
            - name: REDIS_ADDR
              # 使用 K8s 的 DNS 地址连接数据库
              value: "redis-service:6379"
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
            # 为 Backend 添加探针
            readinessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 5
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 15
              periodSeconds: 10
    ```

    应用它：

    ```bash
    kubectl apply -f backend-layer.yaml
    ```

    **检查**：`kubectl get deployment`。

#### 第 3 步：部署 Frontend 层与 Ingress 入口

最后，我们部署面向用户的 Frontend，并用 Ingress 将其暴露到集群外。

1.  **创建 Frontend 的 Manifest 文件**
    创建一个 `frontend-layer.yaml` 文件。我们将使用镜像 `mlabouardy/guester-ui-react:0.1`。

    ```yaml
    # frontend-layer.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend-service
      labels:
        app: my-three-tier-app
        tier: frontend
    spec:
      ports:
      - port: 80
        targetPort: 80
      selector:
        app: my-three-tier-app
        tier: frontend
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: my-three-tier-app
          tier: frontend
      template:
        metadata:
          labels:
            app: my-three-tier-app
            tier: frontend # 关键标签
        spec:
          containers:
          - name: frontend-ui
            image: mlabouardy/guester-ui-react:0.1
            ports:
            - containerPort: 80
            env:
            - name: REACT_APP_API_URL
              # 告知前端 API 服务的地址
              value: "http://localhost/api" # 通过 Ingress 访问
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: main-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$2
    spec:
      rules:
      - http:
          paths:
          - path: /?(.*)
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api/?(.*)
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
    ```

    应用它：

    ```bash
    kubectl apply -f frontend-layer.yaml
    ```

    **检查**：`kubectl get ingress`。

#### 第 4 步：配置 NetworkPolicy 实现安全隔离

现在所有服务都已运行，但网络是全通的，任何一个 Pod 都可以直接访问数据库。这是不安全的。我们来定义严格的访问规则。

创建一个 `network-policies.yaml` 文件：

```yaml
# network-policies.yaml

# 策略1: 隔离数据库，只允许来自 Backend 的流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend

---
# 策略2: 隔离 Backend，只允许 Frontend 和 Ingress Controller 访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    # 允许 Ingress Controller 访问 Backend
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx

---
# 策略3: 隔离 Frontend，只允许来自 Ingress Controller 的流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
```

应用它：

```bash
kubectl apply -f network-policies.yaml
```

#### 第 5 步：最终验证与监控

1.  **访问应用**
    获取 Minikube 的 IP 地址，这将是你的应用入口。

    ```bash
    MINIKUBE_IP=$(minikube ip)
    echo "请在浏览器中打开 http://$MINIKUBE_IP"
    ```

    打开浏览器，你应该能看到留言板的界面。尝试添加几条留言，如果能成功写入和显示，说明从 Frontend -\> Backend -\> Database 的整条链路都已打通！

2.  **验证 NetworkPolicy**
    让我们尝试从一个“无关”的 Pod 直接访问数据库，这应该会失败。

    ```bash
    # 启动一个临时 Pod，尝试连接 Redis (这会因为网络策略而超时失败)
    kubectl run --rm -it --image=redis:6.2-alpine test-pod -- redis-cli -h redis-service -a supersecret PING
    ```

    这个命令会卡住然后超时，因为 `test-pod` 没有 `tier: backend` 标签，被我们的 `db-policy` 拒绝了。**这证明了我们的网络隔离是有效的！**

3.  **使用 `kubectl top` 监控**
    查看我们部署的这一套应用的资源使用情况。

    ```bash
    kubectl top pod
    ```

    你会看到 `frontend`, `backend`, `redis` 等 Pod 的实时 CPU 和内存消耗，这得益于我们开启的 `metrics-server`。

-----

### 🧹 清理环境

这个实验创建了很多资源，我们可以用标签来方便地一次性删除它们。

```bash
# 删除所有带有 app=my-three-tier-app 标签的资源
kubectl delete all,pvc,ingress,secret,networkpolicy -l app=my-three-tier-app

# 如果有残留，可以删除文件创建的所有资源
kubectl delete -f frontend-layer.yaml
kubectl delete -f backend-layer.yaml
kubectl delete -f database-layer.yaml
kubectl delete -f network-policies.yaml
kubectl delete secret redis-secret
```

最后，停止 Minikube：

```bash
minikube stop
```

-----

### 总结

**祝贺你！你已经成功完成了第 21 天的综合实战！**

今天你没有学习任何新命令，但你完成了一项意义非凡的任务：你像一位真正的 DevOps 工程师一样，将过去一周所学的零散知识点串联起来，构建了一个符合现代云原生应用标准的、具备**持久化**、**高可用**、**可观测性**和**安全性**的微服务系统。

回顾一下你今天集成的技术栈：

  * **应用架构**: 三层微服务清晰解耦。
  * **数据持久化**: `StatefulSet` + `PVC` 保证了数据库的稳定和数据不丢失。
  * **配置管理**: `ConfigMap` 和 `Secret` 实现了配置与代码分离，保护了敏感信息。
  * **流量管理**: `Ingress` 提供了统一、灵活的七层路由入口。
  * **安全加固**: `NetworkPolicy` 实现了 Pod 间的“零信任网络”，最小化了攻击面。
  * **应用韧性**: `Liveness` 和 `Readiness` 探针赋予了应用自我检测和恢复的能力。
  * **基础监控**: `Metrics Server` 让你能随时掌握应用的资源消耗。

你已经从学习单个 Kubernetes 对象，成长为能够综合运用它们来解决实际问题。这是质的飞跃。请为你自己鼓掌！继续保持这份热情，你的 K8s 之旅将充满无限可能。