没问题！欢迎来到第 20 天的学习。今天我们将学习一个至关重要的概念，它能让你的应用从“能运行”进化到“能健壮、可靠地运行”。这就是健康检查探针 (Probes)。

如果说监控是让你**被动地**观察应用的健康状况，那么探针就是让 Kubernetes **主动地**、持续地去“问候”你的应用：“你还好吗？能工作了吗？” 从而实现自动化的故障恢复和流量控制。

-----

### 🎯 学习内容：理论知识篇

想象一下，你开了一家新餐厅 (你的 Pod)。

  * 仅仅把餐厅大门打开 (进程启动) 是不够的。
  * 你需要确保厨房里的厨师没有晕倒 (应用没有死锁)，这才能持续做菜。
  * 你还需要确保食材准备好了，烤箱预热完毕了，才能开始接纳客人点餐 (接收流量)。

Kubernetes 的探针就是帮你自动化完成这些检查的“餐厅经理”。

#### 1\. LivenessProbe (存活探针)：厨师还好吗？

  * **回答的问题**：“这个容器还好吗，还是已经陷入了无法恢复的僵尸状态？”
  * **检查什么**：用于检测应用是否已经**死锁**或**崩溃**。一个应用进程可能依然存在，但内部可能因为线程死锁、资源耗尽等问题而无法响应任何请求。
  * **触发的行为**：如果 LivenessProbe 检测失败，Kubelet 会**杀死**这个容器，然后根据其`restartPolicy`（通常是 `Always`）来**重启**它。
  * **现实比喻**：餐厅经理每隔 5 分钟就去厨房看一眼厨师。如果发现厨师晕倒了（LivenessProbe 失败），他会立刻叫救护车拉走，并让备用厨师顶上（重启容器）。这是一种“残酷但有效”的恢复手段。

#### 2\. ReadinessProbe (就绪探针)：餐厅可以接客了吗？

  * **回答的问题**：“这个容器准备好接收新的网络流量了吗？”
  * **检查什么**：用于检测应用是否**准备就绪**可以开始处理请求。有些应用启动时需要加载大量配置、预热缓存或建立数据库连接，这个过程可能需要几十秒甚至几分钟。在这期间，它虽然在运行，但还不能正常工作。
  * **触发的行为**：如果 ReadinessProbe 检测失败，容器**不会被杀死或重启**。相反，它的 IP 地址会从其所属的 Service 的 Endpoints 列表中被**移除**。这样一来，`kube-proxy` 就不会再把新的流量转发给这个尚未就绪的 Pod。当它后续准备就绪，ReadinessProbe 成功后，它会再次被添加回 Endpoints 列表。
  * **现实比喻**：餐厅开门营业了，但经理看到厨房的烤箱还没预热好（ReadinessProbe 失败）。他会立刻在门口挂上“暂停营业”的牌子（从 Service Endpoints 移除），不让新客人进来。等烤箱预热好了（ReadinessProbe 成功），他再把牌子摘掉，开始迎客。

#### 3\. StartupProbe (启动探针)：开业大酬宾准备好了吗？

  * **回答的问题**：“这个启动很慢的应用，最终成功启动了吗？”
  * **检查什么**：专门用于那些启动时间特别长的应用（例如需要几分钟才能启动的 Java 应用）。
  * **触发的行为**：在 StartupProbe 成功之前，LivenessProbe 和 ReadinessProbe 都**不会被启用**。这给了应用足够长的时间来完成它复杂的启动过程。
      * 如果 StartupProbe 在规定的时间内最终成功了，系统就会切换到使用 Liveness/Readiness 探针进行后续的健康检查。
      * 如果 StartupProbe 最终失败了（例如超过了 `failureThreshold * periodSeconds` 的总时长），那么 Kubelet 就会像 LivenessProbe 失败一样，**杀死并重启**容器。
  * **现实比喻**：餐厅搞盛大的开业典礼，经理知道整个准备工作需要 30 分钟。在这 30 分钟内，他不会去检查“厨师是否活着”或“厨房是否能接单”，他只是耐心等待。但如果 30 分钟后，开业准备还没完成（StartupProbe 失败），经理就认为出了大问题，会把所有员工解雇重来（重启容器）。

| 探针类型 | 目的 | 失败后的动作 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **LivenessProbe** | 检测容器是否“活着” | **重启容器** | 应用发生死锁，重启是唯一恢复手段 |
| **ReadinessProbe**| 检测容器是否“准备好” | **从 Service 中移除** | 应用启动中、或临时过载无法处理新请求 |
| **StartupProbe** | 保护启动慢的应用 | **重启容器** (若超时) | 启动时间很长的应用，防止被 Liveness 误杀|

**探针的配置方式**：

  * `httpGet`: 向容器的指定端口和路径发送 HTTP GET 请求，响应码为 2xx 或 3xx 即为成功。
  * `tcpSocket`: 尝试与容器的指定端口建立 TCP 连接，能建立即为成功。
  * `exec`: 在容器内执行一个命令，如果命令退出码为 0 即为成功。

理论已清晰，让我们马上动手，感受 K8s 的“自愈能力”。

-----

### 🎯 实战任务：动手操作篇

为了更好地模拟健康和不健康状态，我们将使用一个专门用于此演示的镜像 `agrimmer/go-http-healthcheck`。

  * 它在 `8080` 端口上运行一个 HTTP 服务。
  * `/healthz` 和 `/readyz` 路径默认返回 `200 OK`。
  * 如果你在容器内执行 `touch /tmp/unhealthy`，`/healthz` 就会开始返回 `503`。
  * 如果你在容器内执行 `touch /tmp/notready`，`/readyz` 就会开始返回 `503`。

#### 第 1 步：LivenessProbe - 观察自动重启

创建一个名为 `liveness-demo.yaml` 的文件：

```yaml
# liveness-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: healthcheck-app
    image: agrimmer/go-http-healthcheck
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      # K8s 第一次探测前等 5 秒
      initialDelaySeconds: 5
      # 每 5 秒探测一次
      periodSeconds: 5
```

1.  **部署应用并观察**

    ```bash
    kubectl apply -f liveness-demo.yaml

    # 使用 -w 参数持续观察 Pod 状态
    kubectl get pod liveness-pod -w
    ```

    你会看到 Pod 很快进入 `Running` 状态，`RESTARTS` 计数为 0。

2.  **模拟服务挂掉**
    打开**一个新的终端**，执行 `exec` 命令进入 Pod 内部，并创建一个文件来让 `/healthz` 端点失败。

    ```bash
    kubectl exec -it liveness-pod -- touch /tmp/unhealthy
    ```

    执行后，你可以用 `curl` 测试一下（可选）：`kubectl exec -it liveness-pod -- curl localhost:8080/healthz`，你会看到 `Service Unavailable`。

3.  **观察 K8s 自动重启**
    回到你**第一个终端**，继续观察 `kubectl get pod liveness-pod -w` 的输出。
    大约 10-15 秒后，你会看到 Pod 的状态发生变化：

    ```
    NAME           READY   STATUS    RESTARTS   AGE
    liveness-pod   1/1     Running   0          35s
    # 模拟故障后...
    liveness-pod   0/1     Running   0          50s   <-- 变为 Not Ready
    liveness-pod   0/1     Running   1          60s   <-- RESTARTS 变为 1
    liveness-pod   1/1     Running   1          70s   <-- 恢复正常
    ```

    **成功了！** `RESTARTS` 计数从 0 变成了 1。这是因为 LivenessProbe 连续几次探测到失败后，Kubernetes 自动杀死了这个“不健康”的容器并重启了它。重启后的新容器里没有 `/tmp/unhealthy` 文件，所以恢复了健康。

#### 第 2 步：ReadinessProbe - 验证流量被切断

首先删除刚才的 Pod：`kubectl delete pod liveness-pod`。

现在创建一个包含 ReadinessProbe 和 Service 的文件 `readiness-demo.yaml`：

```yaml
# readiness-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness-app
spec:
  containers:
  - name: healthcheck-app
    image: agrimmer/go-http-healthcheck
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-service
spec:
  selector:
    app: readiness-app
  ports:
  - port: 80
    targetPort: 8080
```

1.  **部署并验证初始状态**

    ```bash
    kubectl apply -f readiness-demo.yaml
    ```

    查看 Pod 状态，`READY` 列应该是 `1/1`：

    ```bash
    kubectl get pod readiness-pod
    # NAME            READY   STATUS    RESTARTS   AGE
    # readiness-pod   1/1     Running   0          15s
    ```

    查看 Service 的 Endpoints，应该能看到 Pod 的 IP 地址：

    ```bash
    kubectl describe service readiness-service
    # Name:              readiness-service
    # ...
    # Endpoints:         172.17.0.5:8080   <-- Pod IP 在这里
    # ...
    ```

2.  **模拟应用“未就绪”**
    在**另一个终端**中，进入 Pod 并创建 `/tmp/notready` 文件：

    ```bash
    kubectl exec -it readiness-pod -- touch /tmp/notready
    ```

3.  **观察流量入口被移除**
    回到**第一个终端**。
    首先，观察 Pod 状态。`RESTARTS` 计数不会变，但 `READY` 状态会从 `1/1` 变为 `0/1`。

    ```bash
    kubectl get pod readiness-pod -w
    # NAME            READY   STATUS    RESTARTS   AGE
    # readiness-pod   1/1     Running   0          1m
    # ...
    # readiness-pod   0/1     Running   0          1m15s  <-- 状态变化！
    ```

    最关键的一步：再次检查 Service 的 Endpoints：

    ```bash
    kubectl describe service readiness-service
    # Name:              readiness-service
    # ...
    # Endpoints:         <none>            <-- Pod IP 消失了！
    # ...
    ```

    **实验成功！** 这证明了当 ReadinessProbe 失败时，Kubernetes 会自动将该 Pod 从流量入口中移除，以确保不会有用户请求被发送到一个无法处理它的 Pod。

    **恢复**：你可以试着 `kubectl exec -it readiness-pod -- rm /tmp/notready`，稍等片刻后，Pod 会再次变为 `READY`，其 IP 也会重新出现在 Service Endpoints 中。

-----

### 🧹 清理环境

1.  **删除 Pod 和 Service:**
    ```bash
    kubectl delete -f readiness-demo.yaml
    # 如果 liveness-pod 还在，也一并删除
    kubectl delete -f liveness-demo.yaml --ignore-not-found
    ```
2.  **停止 Minikube:**
    ```bash
    minikube stop
    ```

-----

### 总结

恭喜你完成了第 20 天的学习，你已经掌握了构建“打不死”的 resilient 应用的核心技能！

1.  **LivenessProbe** 是应用的\*\*“救心丸”\*\*，在应用死锁时通过重启来强制恢复。
2.  **ReadinessProbe** 是应用的\*\*“迎宾员”\*\*，确保只有在应用完全准备好的情况下才让流量进入。
3.  **StartupProbe** 是给“慢性子”应用的\*\*“耐心启动器”\*\*，防止它们在启动过程中被误杀。

正确配置这三种探针，是任何生产级 Kubernetes 应用的必备条件，它将 Kubernetes 的自动化和自愈能力发挥到了极致。你已经离 Kubernetes 专家又近了一大步！