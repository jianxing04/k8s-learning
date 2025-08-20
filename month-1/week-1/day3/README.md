干得漂亮，欢迎来到 Day 3！昨天我们学习了 Deployment，它能确保我们的应用 Pod 高可用、可扩展。但正如我们最后提到的，这些 Pod 的 IP 地址是动态变化的，我们还无法方便地从外部或内部访问它们。

今天，我们就来解决这个“交通和寻址”问题，主角是 **Service**。

-----

### 🎯 **目标 3：掌握 Service 的作用，理解不同类型**

-----

### **学习内容**

#### **1. Service 解决的问题：Pod IP 不固定**

想象一个场景：Deployment 保证了有 3 个“厨师”（Pod）在同时工作。

  * **问题一：厨师会换班**。一个厨师（Pod）累了（挂了），Deployment 会立刻换上一个新的厨师（新 Pod）。但新厨师的电话（IP 地址）和旧厨师不一样。如果你只记住了旧厨师的电话，你就再也联系不上他了。
  * **问题二：到底联系谁**。有 3 个厨师都在工作，你作为“顾客”（客户端），应该把订单发给谁？难道要记录下所有厨师的电话，然后自己随机选一个吗？

**Service 就是这家餐厅的“统一前台”**。

它的作用是：

1.  **提供一个稳定的入口**：Service 有一个\*\*固定不变的虚拟 IP（称为 ClusterIP）\*\*和 DNS 名称。无论后端的 Pod 如何生生灭灭、IP 如何变化，Service 的这个入口地址永远不变。
2.  **自动发现和负载均衡**：Service 会使用我们在 Deployment 中定义的 **Label Selector** 自动找到它应该管理的所有 Pod（比如所有打了 `app: my-nginx` 标签的 Pod）。当请求到达 Service 时，它会自动将请求转发给其中一个健康的 Pod，实现了负载均衡。

> **小结**：Service 为一组功能相同的 Pod 提供了一个统一、稳定的访问入口，并解决了服务发现和负载均衡的核心问题。

-----

#### **2. ClusterIP：集群内部访问**

这是 **默认的 Service 类型**。

  * **作用**：创建一个仅在**集群内部**可以访问的虚拟 IP。集群外的用户无法访问这个 IP。
  * **工作方式**：Kubernetes 会从内部IP地址池中分配一个 IP 地址给这个 Service。集群内的任何 Pod 都可以通过这个 `ClusterIP` + `端口` 来访问后端的 Pod。
  * **DNS**：更方便的是，每个 Service 都会被自动分配一个 DNS 名称，格式为 `<service-name>.<namespace>.svc.cluster.local`。例如，`default` 命名空间下的 `my-nginx-service` 可以被其他 Pod 直接通过 `http://my-nginx-service` 访问。
  * **适用场景**：绝大多数的微服务间通信。比如，`前端服务` 需要调用 `用户认证服务`，那么 `用户认证服务` 就应该通过 ClusterIP 类型的 Service 来暴露。

-----

#### **3. NodePort：集群外部访问**

有时候我们需要从集群外部访问服务，比如为开发者提供一个测试入口。NodePort 就是一种简单的方式。

  * **作用**：在 ClusterIP 的基础上，额外在集群中\*\*所有节点（Node）\*\*上都打开一个相同的、固定的端口（NodePort）。
  * **工作方式**：
    1.  首先，它会自动创建一个 ClusterIP Service。
    2.  然后，它会在每个 Node 上监听一个端口（默认范围是 30000-32767）。
    3.  任何发送到 `<任意节点IP>:<NodePort>` 的流量，都会被转发到该 Service 的 ClusterIP，再由 ClusterIP 转发给后端的 Pod。
  * **适用场景**：用于开发、测试或演示环境，需要快速将服务暴露给外部网络。
  * **缺点**：
      * 端口号范围受限且不直观（例如 `http://<你的节点IP>:30080`）。
      * 需要额外管理节点 IP 的暴露问题。
      * 不适合用于生产环境的 Web 服务（生产环境通常使用 `LoadBalancer` 或 `Ingress`）。

-----

### **实战任务**

理论学习完毕，让我们把昨天的 Deployment 用 Service 暴露出来！

**准备工作**：昨天的 Deployment 已经被我们删除了，我们先重新创建它。

1.  确保你还有昨天的 `nginx-deployment.yaml` 文件。

2.  执行部署命令：

    `kubectl apply -f nginx-deployment.yaml`

3.  确认 3 个 nginx Pod 正在运行：

    `kubectl get pods`

-----

#### **1. 为 `nginx-deployment` 创建一个 ClusterIP Service**

1.  **创建 YAML 文件**：
    新建一个名为 `nginx-clusterip-service.yaml` 的文件，内容如下：

    ```yaml
    # nginx-clusterip-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx-clusterip
    spec:
      # type: ClusterIP  # 这是默认类型，可以不写
      selector:
        app: my-nginx # **关键**：这个 selector 必须和 Deployment 中 Pod 的 label 完全一致
      ports:
        - protocol: TCP
          port: 80 # Service 自身暴露的端口
          targetPort: 80 # 流量要转发到的目标 Pod 上的端口
    ```

    **关键点解读**：

      * `selector: app: my-nginx`：这行代码告诉 Service：“请去寻找所有 `metadata.labels` 中包含 `app: my-nginx` 的 Pod，并将它们作为你的后端服务实例。”
      * `port: 80`：其他 Pod 访问这个 Service 时使用的端口。
      * `targetPort: 80`：Service 接收到请求后，将流量转发到后端 Pod 容器监听的端口。

2.  **应用 Service 配置**：

    `kubectl apply -f nginx-clusterip-service.yaml`

3.  **查看 Service**：

    `kubectl get service` 或 `kubectl get svc`

    ```bash
    NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP   2d
    my-nginx-clusterip   ClusterIP   10.108.111.22   <none>        80/TCP    15s
    ```

    你已经成功创建了一个 Service，并获得了一个 ClusterIP `10.108.111.22`（你的 IP 会不一样）。

-----

#### **2. 在 Pod 内 curl Service IP，验证负载均衡**

我们无法从自己的电脑（宿主机）上直接访问这个 ClusterIP，因为它只在集群内部有效。为了验证它，我们需要在集群内部启动一个“客户端” Pod。

1.  **启动一个临时的 busybox Pod**：
    我们将使用 `kubectl run` 命令来快速创建一个临时的、可交互的 Pod。

    `kubectl run tmp-client --rm -it --image=busybox -- sh`

      * `--rm`：表示当退出 shell 后，这个 Pod 会被自动删除。
      * `-it`：开启交互式终端。
      * `--image=busybox`：使用一个非常小且包含网络工具的镜像。
      * `-- sh`：进入 Pod 后执行 shell 命令。

    执行后，你的命令行提示符会变成 `#` 或 `$`，表示你已经在这个 `tmp-client` Pod 内部了。

2.  **从 `tmp-client` 内部访问 Service**：
    用 `wget` 或 `curl` 访问我们刚刚创建的 Service 的 ClusterIP（请替换成你自己的 ClusterIP）。

    `wget -qO- 10.108.111.22`

    你会立刻看到熟悉的 Nginx 欢迎页面 HTML！

    ```html
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    ```

3.  **通过 Service 名称访问（更推荐的方式）**：
    在 Pod 内，直接使用 Service 的名字作为域名来访问，更为方便。

    `wget -qO- my-nginx-clusterip`

    同样，你会看到 Nginx 的欢迎页面。这证明了集群内部的 DNS 解析正常工作。在 `tmp-client` Pod 中输入 `exit` 退出并让它自动销毁。

-----

#### **3. 创建 NodePort Service，从宿主机访问**

1.  **创建 YAML 文件**：
    新建 `nginx-nodeport-service.yaml` 文件。它和 ClusterIP 的版本非常相似，只是类型不同。

    ```yaml
    # nginx-nodeport-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx-nodeport
    spec:
      type: NodePort # **关键**：明确指定类型为 NodePort
      selector:
        app: my-nginx
      ports:
        - protocol: TCP
          port: 80 # Service 在集群内部的端口
          targetPort: 80 # Pod 的目标端口
          # nodePort: 30007 # 可以指定一个端口，如果省略，会自动分配一个
    ```

2.  **应用配置**：

    `kubectl apply -f nginx-nodeport-service.yaml`

3.  **查看 Service 并找到 NodePort**：

    `kubectl get svc my-nginx-nodeport`

    ```bash
    NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    my-nginx-nodeport   NodePort   10.101.12.123   <none>        80:31978/TCP   8s
    ```

    注意 `PORT(S)` 字段变成了 `80:31978/TCP`。这表示：

      * 集群内部依然可以通过 `10.101.12.123:80` 访问。
      * 集群外部可以通过 `<任何一个节点的IP>:31978` 访问。

4.  **从宿主机访问**：

      * **方法一 (Minikube 快捷方式)**：如果使用 Minikube，它提供了一个非常方便的命令。

        `minikube service my-nginx-nodeport`

        这个命令会自动在你的默认浏览器中打开 Nginx 的欢迎页面！

      * **方法二 (通用方式)**：
        首先，获取 Minikube 虚拟机（也就是我们的 Kubernetes Node）的 IP 地址。

        `minikube ip`

        假设得到的 IP 是 `192.168.49.2`。然后，使用这个 IP 加上我们刚才查到的 NodePort `31978` 来访问。

        在你的电脑终端里执行：

        `curl http://$(minikube ip):31978`

        或者直接在浏览器地址栏输入 `http://192.168.49.2:31978`。你同样会看到 Nginx 的欢迎页面！

-----

### **清理环境**

删除今天创建的所有资源：

```bash
kubectl delete deployment my-nginx-deployment
kubectl delete service my-nginx-clusterip
kubectl delete service my-nginx-nodeport
```

-----

### **本日总结**

今天你解锁了 Kubernetes 的网络核心——Service！

1.  **理解了 Service 的核心价值**：为 Pod 提供了稳定的访问入口，并内置负载均衡。
2.  **掌握了 ClusterIP Service**：实现了集群内部服务之间的可靠通信。
3.  **掌握了 NodePort Service**：打通了从集群外部到内部服务的简单通道。

我们已经一步步建立了一个高可用、可更新、并且可以从内外访问的应用。但 NodePort 的端口号奇怪，而且直接暴露节点 IP 也不太安全和优雅。如果我们想使用标准的 80/443 端口，并且通过域名（如 `my-app.com`）来访问我们的服务，该怎么办呢？

这就是我们 Day 4 要学习的更高级的网络路由工具：**Ingress**。敬请期待！