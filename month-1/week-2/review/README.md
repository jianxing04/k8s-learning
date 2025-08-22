没问题！下面是**可直接上手的“一条龙实验步骤 + 指令清单”**。按顺序操作即可把第二周涉及的所有知识点（ConfigMap/Secret、PV/PVC、StatefulSet、调度（亲和/污点容忍）、Requests/Limits、HPA、以及验证方法）一遍走完。

> 假设你在 **minikube** 上操作，shell 为 Bash（WSL2 / macOS / Linux 均可）。命名空间统一用 `cn-recap-week2`。

---

# 0) 前置准备

```bash
# 启动或切换到 minikube
minikube start

# 启用 metrics-server（HPA 必备）
minikube addons enable metrics-server

# 创建命名空间
kubectl create ns cn-recap-week2
```

---

# 1) 配置中心：ConfigMap & Secret

## 1.1 创建配置与密文

新建 `01-config-and-secret.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: cn-recap-week2
data:
  REDIS_HOST: "redis-0.redis"   # StatefulSet 首个 Pod 的稳定域名
  REDIS_PORT: "6379"
  APP_MODE: "production"

---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: cn-recap-week2
type: Opaque
data:
  # base64("redispass") -> cmVkaXNwYXNz
  REDIS_PASSWORD: cmVkaXNwYXNz
```

应用：

```bash
kubectl apply -f 01-config-and-secret.yaml
```

**关注点**

* Secret 只是 Base64 编码，不是加密；真正的安全靠 RBAC/加密存储等。
* 我们将用 **envFrom** 把它们注入到前端 Deployment（演示“配置解耦”）。

---

# 2) 有状态服务：Redis StatefulSet + 持久化 PVC

## 2.1 Headless Service（稳定 DNS）

新建 `02-redis-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: cn-recap-week2
labels:
  app: redis
spec:
  clusterIP: None        # Headless
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
```

## 2.2 StatefulSet（每 Pod 独立 PVC、有序部署）

新建 `03-redis-statefulset.yaml`：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: cn-recap-week2
spec:
  serviceName: "redis"
  replicas: 3
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
        image: redis:7
        # 通过 sh -c 读取环境变量（requirepass）
        command: ["sh", "-c", "exec redis-server --appendonly yes --requirepass \"$REDIS_PASSWORD\""]
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: REDIS_PASSWORD
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

应用并观察：

```bash
kubectl apply -f 02-redis-svc.yaml
kubectl apply -f 03-redis-statefulset.yaml

kubectl get pods -n cn-recap-week2 -w
```

**关注点**

* Pod 名称与 DNS：`redis-0.redis`, `redis-1.redis` …
* 每个 Pod 绑定 **独立 PVC**：`data-redis-0` / `data-redis-1` / `data-redis-2`。
* 有序部署与伸缩：按序号 0→1→2 创建/删除。

## 2.3 连通性小测试（可选）

用 `redis` 官方镜像当“客户端”：

```bash
kubectl run redis-client -n cn-recap-week2 --rm -it --image=redis:7 -- bash -lc '
  apt-get update >/dev/null 2>&1 || true
  echo "PING redis-0..."
  redis-cli -h redis-0.redis -a redispass PING
'
```

看到 `PONG` 即 OK。

---

# 3) 调度策略：给节点打标签 + 污点

> 在单节点 minikube 上也可以演示：我们给节点加 “ssd” 标签，并加一个 **PreferNoSchedule** 污点（不会强制阻止其他 Pod，但会“尽量避免”调度没有容忍的 Pod）。

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node "$NODE" disktype=ssd --overwrite
kubectl taint node "$NODE" dedicated=frontend:PreferNoSchedule --overwrite
kubectl get node "$NODE" --show-labels
kubectl describe node "$NODE" | grep -i taints -A1
```

**关注点**

* **NodeAffinity（required）** 会强制前端 Pod 只去有 `disktype=ssd` 的节点。
* **污点/容忍**：我们给节点加污点，前端 Pod 有对应容忍（可调度；其他无容忍的 Pod 会“尽量”避开）。

---

# 4) 无状态前端：Deployment（含 envFrom、Requests/Limits、Affinity、Tolerations）

这里用 `nginx` 作为前端“占位”，重点是演示配置注入、调度策略、资源配额、HPA 伸缩目标等。

新建 `04-frontend-and-svc.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: cn-recap-week2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "frontend"
        effect: "PreferNoSchedule"
      containers:
      - name: web
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: redis-secret
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

---
apiVersion: v1
kind: Service
metadata:
  name: guestbook-service
  namespace: cn-recap-week2
spec:
  type: NodePort
  selector:
    app: guestbook
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080   # 也可省略让集群分配（范围 30000-32767）
```

应用并验证：

```bash
kubectl apply -f 04-frontend-and-svc.yaml
kubectl get deploy/pod/svc -n cn-recap-week2 -o wide
```

访问（任选其一）：

```bash
# 方式A：NodePort（推荐）
minikube service guestbook-service -n cn-recap-week2 --url

# 方式B：直接 curl NodeIP:30080
minikube ip
curl http://$(minikube ip):30080
```

**关注点**

* `envFrom` 已注入了 `REDIS_*`、`APP_MODE`，但 **环境变量是静态的**（更新 ConfigMap/Secret 后需要重启 Pod 才会生效）。
* Requests/Limits 直接决定了 HPA 的利用率计算分母与 QoS 等级。
* NodeAffinity 与 Tolerations 已生效（查看 `kubectl describe pod`）。

---

# 5) HPA：按 CPU 自动伸缩

新建 `05-hpa.yaml`：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: guestbook-hpa
  namespace: cn-recap-week2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: guestbook
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

应用并观察：

```bash
kubectl apply -f 05-hpa.yaml
kubectl get hpa -n cn-recap-week2
```

---

# 6) 产生负载，观察 HPA 扩缩

> 用集群内的“压测生成器”循环请求前端 Service。

```bash
kubectl run load-generator -n cn-recap-week2 --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://guestbook-service.cn-recap-week2.svc.cluster.local >/dev/null; done'
```

实时观察：

```bash
# 观察 HPA 指标变化与副本数调整
kubectl get hpa -n cn-recap-week2 -w

# 观察 Deployment/Pod 副本变化
kubectl get deploy/pod -n cn-recap-week2 -w

# 查看实时资源指标（metrics-server）
kubectl top pods -n cn-recap-week2
kubectl top nodes
```

**关注点**

* HPA 需要 **metrics-server** 正常工作；
* HPA 的 CPU 利用率 = 实际使用 / `requests.cpu`；
* 负载足够时副本会从 2 增长（最多 6），负载降低后会缩回。

> 如果负载不够触发扩容：
>
> * 将 `averageUtilization` 调低，比如 20；
> * 或将 `requests.cpu` 减小到 `50m`；
> * 或多开几个 load-generator：
>
>   ```bash
>   for i in {1..5}; do
>     kubectl run "load-generator-$i" -n cn-recap-week2 --image=busybox:1.36 --restart=Never -- \
>       sh -c 'while true; do wget -q -O- http://guestbook-service.cn-recap-week2.svc.cluster.local >/dev/null; done' &
>   done
>   ```

---

# 7) 验证“配置解耦”和“热更新对比”

## 7.1 更新 ConfigMap（环境变量注入场景）

```bash
# 修改 APP_MODE
kubectl -n cn-recap-week2 patch configmap app-config \
  --type merge -p '{"data":{"APP_MODE":"staging"}}'

# 因为前端用 envFrom 注入，已运行 Pod **不会自动**拿到新值
# 需要滚动重启让 Pod 重新注入环境变量：
kubectl -n cn-recap-week2 rollout restart deployment/guestbook
```

**关注点**

* 使用 **环境变量** 注入时，更新需要 **重启 Pod** 才生效（静态）。
* 若使用 **ConfigMap Volume** 文件挂载，并且应用支持热加载，可不重启即“动态生效”。

---

# 8) 验证 StatefulSet 的“状态稳定性”

## 8.1 写入数据

```bash
kubectl run redis-client -n cn-recap-week2 --rm -it --image=redis:7 -- bash -lc '
  echo "SET key1 hello"
  redis-cli -h redis-0.redis -a redispass SET key1 hello
  echo "GET key1:"
  redis-cli -h redis-0.redis -a redispass GET key1
'
```

## 8.2 删除 `redis-0` Pod，看数据是否还在（PVC 保留）

```bash
kubectl delete pod redis-0 -n cn-recap-week2
kubectl get pods -n cn-recap-week2 -w

# 待其重建 Running 后再次读取
kubectl run redis-client -n cn-recap-week2 --rm -it --image=redis:7 -- bash -lc '
  echo "GET key1 after redis-0 recreated:"
  redis-cli -h redis-0.redis -a redispass GET key1
'
```

**关注点**

* Pod 被删除后，**绑定的 PVC 不会被删除**，数据仍在；
* 这就是 StatefulSet “每个 Pod 有独立身份与独立存储”的核心价值。

---

# 9) 验证调度策略

## 9.1 查看前端 Pod 的节点安排

```bash
kubectl get pods -n cn-recap-week2 -o wide -l app=guestbook
kubectl describe pod -n cn-recap-week2 -l app=guestbook | grep -i -E "Node:|Tolerations:|nodeAffinity" -A3
```

**关注点**

* Pod 被调度到具有 `disktype=ssd` 的节点；
* 污点 `dedicated=frontend:PreferNoSchedule` 已被容忍（Tolerations）。

---

# 10) 清理实验（可选）

```bash
# 删除命名空间（会清理其中所有对象，包括 PVC/PV）
kubectl delete ns cn-recap-week2

# 移除节点标签与污点
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node "$NODE" disktype-   # 去掉标签
kubectl taint node "$NODE" dedicated-  # 去掉污点
```

---

## ✅ 你复盘到的知识点，在本实验如何“直观出现”

* **ConfigMap/Secret**：前端用 `envFrom` 注入；演示“更新后需要重启”。
* **PV/PVC**：`volumeClaimTemplates` 为每个 Redis Pod 自动创建独立 PVC。
* **StatefulSet**：稳定 DNS（`redis-0.redis`）、有序部署、删除 Pod 数据不丢。
* **调度策略**：节点打标签 + NodeAffinity（required）；污点 + Tolerations（PreferNoSchedule）。
* **Requests/Limits**：前端 `100m/500m` & `128Mi/256Mi`，为 HPA 与 QoS 提供基础。
* **HPA**：基于 CPU 利用率（目标 50%）自动扩缩；通过内置压测 Pod 施压观察扩容。

如果你愿意，我可以把上述 **所有 YAML** 合并成一个 `all-in-one.yaml`（分段 `---`），你只需要 `kubectl apply -f all-in-one.yaml` 一次跑起；或者给你做一份 **打印版实验手册（可发给同学/保存复习）**。
