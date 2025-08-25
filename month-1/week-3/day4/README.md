好的，我们准时开始第 18 天的学习！今天的主题是日志管理。日志是应用的“眼睛”，当出现问题时，它们是我们排查故障最重要的线索。在 Kubernetes 这样分布式的环境中，如何有效管理日志至关重要。

让我们一起揭开 K8s 日志管理的奥秘。

-----

### 🎯 学习内容：理论知识篇

#### 1\. Kubernetes 日志的基础：标准输出 (stdout) 和标准错误 (stderr)

在 Kubernetes (或者说，在容器化的世界) 中，最佳实践是让应用程序**不要将日志写入到文件**，而是直接将它们输出到**标准输出 (`stdout`)** 和**标准错误 (`stderr`)** 流。

为什么？

  * **容器的无状态性**：容器的文件系统是短暂的。如果容器崩溃或被删除，里面的日志文件也会随之消失。
  * **简单高效**：容器运行时（如 Docker 或 containerd）会自动捕获这两个流。Kubernetes 就能很方便地通过底层容器运行时来收集这些日志，而无需进入容器内部去读取特定文件。

`kubectl logs` 命令就是利用这一机制来工作的。它实际上是请求 Kubelet 从节点上获取由容器运行时捕获的特定容器的 `stdout` 和 `stderr` 日志。

#### 2\. `kubectl logs`：你的第一日志工具

这是每个 K8s 工程师最常用的调试命令之一。它允许你直接查看一个 Pod 中某个容器的日志。

  * **基本用法**: `kubectl logs <pod-name>`
  * **查看特定容器**: 如果 Pod 中有多个容器，需要指定容器名 `kubectl logs <pod-name> -c <container-name>`
  * **实时追踪 (tailing)**: 使用 `-f` 或 `--follow` 参数，可以像 `tail -f` 一样实时查看新产生的日志，这对于观察实时动态非常有用。

然而，`kubectl logs` 只适用于临时性的调试，它有明显的局限性：

  * **日志是临时的**：Pod 被删除后，日志就没了。
  * **无法聚合**：当你的应用有多个副本（Replica）时，请求可能被路由到任何一个副本。如果出现问题，你可能需要逐个检查每个 Pod 的日志，非常低效。
  * **缺乏高级功能**：无法进行复杂的搜索、过滤、聚合分析或可视化。

#### 3\. 多副本 Pod 日志查看方法

正如上面提到的，逐个检查多副本 Pod 的日志是不可行的。`kubectl` 提供了一个稍好一点的方法：使用**标签选择器 (`-l`)**。

`kubectl logs -f -l <label-key>=<label-value>`

这个命令会同时从所有匹配该标签的 Pod 中流式传输日志。输出的每一行都会有 Pod 名称作为前缀，让你能够区分日志的来源。这对于同时观察一个 Deployment 下所有 Pod 的行为非常方便，也是我们今天实战的核心。

#### 4\. 集中式日志：生产环境的解决方案

为了解决 `kubectl logs` 的局限性，生产环境中我们都需要一套集中式日志系统。它的核心思想是：在每个节点上部署一个**日志代理 (Log Agent)**，由它自动收集该节点上所有容器的日志，并将它们发送到一个统一的、可持久化存储和查询的**日志中心**。

目前最主流的两大方案是 EFK 和 Loki。

**A. EFK 体系 (Elasticsearch + Fluentd + Kibana)**

这是传统且功能非常强大的日志解决方案。

  * **E - Elasticsearch**: 一个强大的搜索引擎和分析引擎。所有日志都被发送到这里进行存储和索引。它会对日志的**全文进行索引**，提供极其强大的搜索能力。
  * **F - Fluentd**: 日志代理。它作为一个 DaemonSet 部署在每个 Kubernetes 节点上，负责收集容器日志，进行一些简单的处理（如添加 K8s 元数据），然后转发给 Elasticsearch。
  * **K - Kibana**: 一个 Web UI，用于可视化和查询 Elasticsearch 中的数据。你可以用它来创建漂亮的日志仪表盘、搜索特定错误等。

**优点**: 功能极其强大，生态成熟。
**缺点**: 资源消耗大（特别是 Elasticsearch 对内存和磁盘要求高），部署维护相对复杂。

**B. Loki + Grafana + Promtail 体系**

这是一个更新、更轻量级的云原生日志解决方案，由 Grafana Labs 开发。

  * **Loki**: 日志存储后端。它的核心设计理念是**只对元数据（标签）进行索引，而不是全文**。例如，它会索引 `pod="nginx-123"`，`namespace="prod"` 这些标签，但不会索引日志内容本身 "GET / HTTP/1.1 200"。
  * **Promtail**: 日志代理。专门为 Loki 设计的代理，非常轻量。它负责收集日志，并从 K8s API 获取元数据，将它们作为标签附加到日志流上，然后发送给 Loki。
  * **Grafana**: 可视化前端。你可能已经用它来展示 Prometheus 的监控指标了。Grafana 从 7.0 版本开始深度集成了 Loki，你可以在同一个仪表盘上无缝地关联指标（如 CPU 升高）和相关的日志。

**优点**: 资源占用小，存储成本低，与 Prometheus/Grafana 生态无缝集成。
**缺点**: 搜索能力不如 Elasticsearch 灵活（因为不索引全文）。

理论学习结束，我们来动手实践一下 `kubectl logs` 的强大功能。

-----

### 🎯 实战任务：动手操作篇

#### 第 1 步：部署一个多副本 Nginx Deployment

创建一个名为 `nginx-log-demo.yaml` 的文件，并写入以下内容。我们部署 3 个 Nginx 副本。

```yaml
# nginx-log-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        # 这个标签是关键，我们稍后会用它来选择所有副本
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: my-nginx
  ports:
  - port: 80
    targetPort: 80
  # 使用 NodePort 以便从外部访问来产生日志
  type: NodePort
```

应用这个文件来创建资源：

```bash
kubectl apply -f nginx-log-demo.yaml
```

**验证部署:**

```bash
kubectl get pods -l app=my-nginx
```

你应该能看到 3 个状态为 `Running` 的 Nginx Pod。

#### 第 2 步：产生访问日志

Nginx 默认会记录访问日志。为了看到日志，我们需要向它发送一些 HTTP 请求。

首先，获取 Minikube IP 和 Nginx Service 的 NodePort：

```bash
MINIKUBE_IP=$(minikube ip)
NGINX_PORT=$(kubectl get svc nginx-service -o=jsonpath='{.spec.ports[0].nodePort}')

echo "访问地址: http://$MINIKUBE_IP:$NGINX_PORT"
```

现在，打开**一个新的终端**，使用一个循环 `curl` 命令来持续发送请求，这样我们就能在另一个终端观察到实时日志了。

```bash
# 在新终端中运行此命令
while true; do curl http://$MINIKUBE_IP:$NGINX_PORT &> /dev/null; echo "Request sent"; sleep 2; done
```

这个命令会每 2 秒发送一次请求。

#### 第 3 步：使用 `kubectl logs` 查看日志

回到你**原来的终端**。

1.  **查看单个 Pod 日志**
    首先，获取一个 Pod 的名字：

    ```bash
    # 获取第一个 pod 的名字
    POD_NAME=$(kubectl get pods -l app=my-nginx -o jsonpath='{.items[0].metadata.name}')
    echo "正在查看 Pod: $POD_NAME 的日志"

    # 查看该 Pod 的所有当前日志
    kubectl logs $POD_NAME
    ```

    你会看到这个 Pod 累计的访问记录。

2.  **实时追踪单个 Pod 日志**
    现在使用 `-f` 参数来实时追踪。

    ```bash
    kubectl logs -f $POD_NAME
    ```

    你会看到，每当另一个终端发送请求并恰好被这个 Pod 处理时，这里就会打印出一条新的日志。按 `Ctrl+C` 退出。

3.  **同时追踪所有副本的日志（核心任务）**
    这是最有用的功能。我们使用标签选择器 `-l app=my-nginx`。

    ```bash
    echo "正在同时追踪所有 app=my-nginx 的 Pod 日志..."
    kubectl logs -f -l app=my-nginx
    ```

    现在，你会看到来自不同 Pod 的日志交错出现。`kubectl` 很贴心地为每一行日志都加上了来源 Pod 的名称作为前缀，例如：

    ```
    [nginx-deployment-7d5dcfd6d8-abcde] 172.17.0.1 - - [25/Aug/2025:07:42:10 +0000] "GET / HTTP/1.1" 200 ...
    [nginx-deployment-7d5dcfd6d8-fghij] 172.17.0.1 - - [25/Aug/2025:07:42:12 +0000] "GET / HTTP/1.1" 200 ...
    [nginx-deployment-7d5dcfd6d8-klmno] 172.17.0.1 - - [25/Aug/2025:07:42:14 +0000] "GET / HTTP/1.1" 200 ...
    ```

    这清晰地展示了 K8s 的 Service 是如何将流量负载均衡到不同 Pod 上的。按 `Ctrl+C` 退出。

#### 第 4 步（可选）：了解 Minikube 的 EFK 插件

Minikube 提供了一个简单的 EFK 插件，可以让你体验一下集中式日志系统。

**注意**：这个插件相当消耗资源，可能会让你的电脑变慢。

```bash
# 启用 EFK 插件
minikube addons enable efk

# 等待几分钟让所有 EFK 相关的 Pod 启动
# 你可以观察它们的状态
kubectl get pods -n kube-system | grep "elasticsearch\|fluentd\|kibana"

# 启动完成后，可以通过端口转发来访问 Kibana UI
echo "等待 Kibana Pod 启动..."
kubectl wait --for=condition=ready pod -l app=kibana -n kube-system --timeout=300s

echo "正在转发 Kibana 端口到本地 5601..."
kubectl port-forward -n kube-system svc/kibana-logging 5601:5601
```

现在，在浏览器中打开 `http://localhost:5601`，你就能看到 Kibana 的界面。你可能需要创建一个 index pattern (索引模式)，通常使用 `logstash-*` 就可以匹配到由 Fluentd 收集来的日志。之后，你就可以在 "Discover" 页面看到并搜索你的 Nginx 访问日志了。

体验结束后，记得关闭 `kubectl port-forward` (`Ctrl+C`)。

-----

### 🧹 清理环境

1.  别忘了停止在另一个终端运行的 `while` 循环 (`Ctrl+C`)。
2.  **删除 Nginx 应用:**
    ```bash
    kubectl delete -f nginx-log-demo.yaml
    ```
3.  **（如果启用了）禁用 EFK 插件:**
    ```bash
    minikube addons disable efk
    ```
4.  **停止 Minikube:**
    ```bash
    minikube stop
    ```

-----

### 总结

祝贺你完成第 18 天的学习！今天你掌握了 K8s 中日志处理的关键知识：

1.  **基础机制**：理解了日志应输出到 `stdout`/`stderr`。
2.  **日常调试**：熟练运用 `kubectl logs` 及其 `-f` 和 `-l` 参数，能够高效地查看单个或多个 Pod 的日志。
3.  **生产理念**：对 EFK 和 Loki 两大主流集中式日志方案的工作原理有了清晰的认识，为将来构建生产级可观测性系统打下了基础。

日志是排查问题的生命线。掌握了今天的技能，你在诊断和解决 K8s 应用问题时会更加得心应手！继续前进！