太棒了，欢迎来到第 13 天，也是我们第二周学习的综合实战日！

在过去的一周里，你已经掌握了 Kubernetes 中更高级、更贴近生产实践的概念：

  * **Day 8**: `ConfigMap` & `Secret` (配置与凭证管理)
  * **Day 9**: `Volume` & `PVC` (持久化存储)
  * **Day 10**: `StatefulSet` (有状态应用部署)
  * **Day 11**: `Scheduler` & `NodeSelector` (调度控制)
  * **Day 12**: `Resources` & `HPA` (资源管理与弹性伸缩)

今天，我们将把所有这些“高级积木”组合起来，搭建一个包含前端 Web 服务、后端数据库、外部化配置、持久化存储和自动扩缩容能力的\*\*“迷你生产级”应用\*\*。

-----

### 🎯 **目标 13：综合运用知识，构建多组件应用**

-----

### **迷你项目架构**

我们将部署一个经典的“留言板” (Guestbook) 应用。它的架构如下：

1.  **前端 (Frontend)**: 一个 Web 应用，我们将用 `Deployment` 管理。

      * 它通过 `ConfigMap` 读取问候语等配置。
      * 它通过 `Secret` 读取数据库密码。
      * 它将由 `HPA` 监控，实现自动扩缩容。
      * 它通过 `NodePort Service` 对外提供访问。

2.  **后端 (Backend)**: 一个 Redis 数据库，我们将用 `StatefulSet` 管理。

      * 主节点 (master) 用于读写留言数据。
      * 它使用 `PVC` 来持久化存储留言板数据。
      * 它通过 `Headless Service` 为集群内部提供稳定的网络标识。

**流量和数据流:**
`用户` -\> `NodePort Service` -\> `Web Pod (Deployment)` -\> `Redis Service` -\> `Redis Pod (StatefulSet)` -\> `PVC`

这是一个非常典型的微服务架构，让我们一步步把它搭建起来！

-----

### **实战任务：一步步构建留言板应用**

#### **第 1 步：配置先行 (ConfigMap & Secret)**

在部署应用之前，我们先创建它所依赖的配置和凭证。

1.  **创建 ConfigMap**:
    我们将用它来定义一个自定义的问候语。
    新建 `guestbook-configmap.yaml`:

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: guestbook-config
    data:
      GREETING_MESSAGE: "Welcome to the K8s Guestbook!"
      APP_MODE: "production"
    ```

    应用它: `kubectl apply -f guestbook-configmap.yaml`

2.  **创建 Secret**:
    我们将用它来存储 Redis 的密码。这次我们换一种方式，使用 `stringData` 字段，这样我们就不需要手动进行 Base64 编码，Kubernetes 会帮我们完成。
    新建 `redis-secret.yaml`:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: redis-secret
    type: Opaque
    stringData:
      # 在这里直接写明文，Kubernetes 保存时会自动编码
      password: "ThisIsAReallyStrongPassword123"
    ```

    应用它: `kubectl apply -f redis-secret.yaml`

-----

#### **第 2 步：部署有状态后端 (Redis StatefulSet)**

现在，我们来部署作为数据存储的 Redis。

1.  **创建 Redis 的 Headless Service**:
    新建 `redis-headless-service.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-svc
    spec:
      clusterIP: None
      selector:
        app: redis
      ports:
      - port: 6379
        targetPort: 6379
    ```

    应用它: `kubectl apply -f redis-headless-service.yaml`

2.  **创建 Redis 的 StatefulSet**:
    我们只部署一个主节点，但依然使用 StatefulSet 以展示其特性。
    新建 `redis-statefulset.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: redis
    spec:
      serviceName: "redis-svc"
      replicas: 1
      selector:
        matchLabels:
          app: redis
      template:
        metadata:
          labels:
            app: redis
        spec:
          containers:
          - name: redis
            image: redis:6.2-alpine
            command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"]
            ports:
            - containerPort: 6379
            env: # 从 Secret 中读取密码作为环境变量
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
            volumeMounts:
            - name: data
              mountPath: /data
      volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi
    ```

    应用它: `kubectl apply -f redis-statefulset.yaml`
    检查 Pod 和 PVC 是否已创建并绑定: `kubectl get pods,pvc -l app=redis`

-----

#### **第 3 步：部署无状态前端 (Web App Deployment)**

后端就绪，现在部署我们的前端 Web 应用。

1.  **创建 Web App 的 NodePort Service**:
    新建 `guestbook-service.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: guestbook-svc
    spec:
      type: NodePort
      selector:
        app: guestbook
      ports:
      - port: 80
        targetPort: 80
    ```

    应用它: `kubectl apply -f guestbook-service.yaml`

2.  **创建 Web App 的 Deployment**:
    我们将使用一个官方的 Go 语言留言板示例应用。
    新建 `guestbook-deployment.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: guestbook-deployment
    spec:
      replicas: 2 # 启动2个副本
      selector:
        matchLabels:
          app: guestbook
      template:
        metadata:
          labels:
            app: guestbook
        spec:
          containers:
          - name: app
            image: gcr.io/google-samples/gb-frontend:v4 # Google 官方示例镜像
            env:
            - name: "GET_HOSTS_FROM"
              value: "dns"
            - name: "REDIS_MASTER_SERVICE_HOST"
              value: "redis-svc.default.svc.cluster.local" # 通过 Headless Service DNS 连接 Redis
            # 从 ConfigMap 注入所有配置为环境变量
            envFrom:
            - configMapRef:
                name: guestbook-config
            resources: # **为 HPA 设置 requests**
              requests:
                cpu: 100m
                memory: 64Mi
              limits:
                cpu: 200m
                memory: 128Mi
            ports:
            - containerPort: 80
    ```

    > **注意**：这个示例镜像没有直接读取 `REDIS_PASSWORD` 的逻辑，在真实项目中，你会把 Secret 注入到环境变量中，然后在应用代码里读取并使用它来连接 Redis。

    应用它: `kubectl apply -f guestbook-deployment.yaml`

-----

#### **第 4 步：配置自动扩缩容 (HPA)**

1.  **确保 Metrics Server 已启用**:
    `minikube addons enable metrics-server`

2.  **为 Web App Deployment 创建 HPA**:
    我们将目标 CPU 设置得低一些 (10%)，以便于触发扩容。
    `kubectl autoscale deployment guestbook-deployment --cpu-percent=10 --min=2 --max=5`

    检查 HPA: `kubectl get hpa`

-----

#### **第 5 步：见证一切！(验证与测试)**

1.  **访问应用**:
    Minikube 提供了方便的命令来获取 NodePort 服务的 URL。
    `minikube service guestbook-svc`
    这个命令会自动在你的浏览器中打开留言板页面。你应该能看到来自 ConfigMap 的问候语 "Welcome to the K8s Guestbook\!"

2.  **测试数据持久化**:

      * 在留言板中随便输入几条消息，点击 "Submit"。
      * 消息会显示在下方列表，这证明前端成功连接到了 Redis 并写入了数据。
      * 现在，我们来模拟 Redis Pod 故障：`kubectl delete pod redis-0`
      * StatefulSet 会立刻重建一个**同名**的 `redis-0` Pod，并**重新挂载**它原来的 PVC。
      * 等待新 Pod 运行后，刷新浏览器页面。你会发现，你之前提交的留言**依然存在**！数据持久化成功！

3.  **测试自动扩缩容**:

      * 打开一个终端，监控 HPA: `kubectl get hpa -w`
      * 打开另一个终端，监控 Pod: `kubectl get pods -w`
      * 打开第三个终端，创建压力产生器：
        `kubectl run -it --rm load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://guestbook-svc; done"`
      * 观察 HPA 监控终端，你会看到 CPU 利用率 (`TARGETS`) 飙升，`REPLICAS` 数量从 2 开始向上增加，最多到 5。同时，Pod 监控终端里会出现新的 `guestbook-deployment` Pod。
      * 停止压力产生器 (Ctrl+C)，等待几分钟，观察副本数自动降回到 2。

-----

### **清理环境**

```bash
# 删除 HPA
kubectl delete hpa guestbook-deployment
# 删除 Deployment 和 Service
kubectl delete deployment guestbook-deployment
kubectl delete service guestbook-svc
# 删除 StatefulSet 和 Service
kubectl delete statefulset redis
kubectl delete service redis-svc
# 删除 PVC (重要，不会自动删除)
kubectl delete pvc data-redis-0
# 删除 ConfigMap 和 Secret
kubectl delete configmap guestbook-config
kubectl delete secret redis-secret
```

-----

### **本日总结**

**为你鼓掌！你今天完成了一个了不起的里程碑！**

你不再是简单地运行单个组件，而是成功地编排了一个包含多个移动部件、具备生产环境雏形的完整应用系统。你综合运用了：

  * **Deployment** 来管理无状态前端。
  * **StatefulSet** 配合 **PVC** 来管理有状态后端。
  * **Service** (NodePort 和 Headless) 来连接一切并暴露服务。
  * **ConfigMap** 和 **Secret** 来实现配置与代码的分离。
  * **HPA** 配合**资源配额**来赋予应用弹性。

你已经建立起了对 Kubernetes 核心生态的全局视野。接下来的学习，可以是在这些核心概念之上，探索更广阔的领域，比如应用打包与分发 (Helm)、服务网格 (Istio)、CI/CD 集成等等。你已经为此打下了坚实的基础！