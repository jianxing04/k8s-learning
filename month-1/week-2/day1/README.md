非常好，欢迎来到 Day 8，也是我们学习新阶段的开始！

在上周，我们成功地部署、管理和暴露了应用。但你可能已经注意到，所有的配置（比如 `http-echo` 的返回文本）都是直接硬编码在 Deployment YAML 文件中的。这在真实世界中是不可行的，因为：

  * **不灵活**：如果想修改一个配置，就必须修改整个 Deployment 文件并重新部署。
  * **不安全**：如果配置中包含密码或 API 密钥，将它们以明文形式存储在代码仓库中是极大的安全风险。
  * **难管理**：同一个应用在开发、测试、生产环境中的配置都不同，硬编码会造成管理混乱。

今天，我们就来学习 Kubernetes 提供的标准解决方案：**ConfigMap** 和 **Secret**，它们能将配置信息从应用代码和部署描述中彻底解耦。

-----

### 🎯 **目标 8：理解如何将配置信息与应用解耦**

-----

### **学习内容**

#### **1. ConfigMap：存储非敏感配置**

可以把 ConfigMap 想象成一个\*\*“公开的配置公告板”\*\*。它是一个 Kubernetes 资源，专门用来存储非敏感的、键值对形式的配置数据。

  * **存储内容**：
      * 单个配置项，如：`app.theme=dark`, `app.feature.enabled=true`
      * 完整的配置文件内容，如整个 `nginx.conf` 或 `application.properties` 文件。
  * **特点**：
      * 以明文形式存储和查看。
      * 大小限制约为 1MB。
      * 适用于环境变量、命令行参数、配置文件等场景。

#### **2. Secret：存储敏感信息**

可以把 Secret 想象成一个\*\*“不透明的密封信封”\*\*。它在结构上和 ConfigMap 非常相似（也是键值对），但专门用于存储敏感信息。

  * **存储内容**：
      * 数据库密码
      * API 密钥或 Token
      * TLS 证书和私钥
  * **特点**：
      * 数据以 **Base64** 编码形式存储。
      * **重要**：Base64 **不是加密**，它只是一种编码方式，可以被轻易地解码回原文。它的主要目的是为了安全地传输和处理包含特殊字符或二进制的数据。
      * Secret 的真正安全性来自于 Kubernetes 的 **RBAC (基于角色的访问控制)**，它可以精确控制哪些用户或 Pod 有权限读取某个 Secret。在生产环境中，还会配合 etcd 加密、Vault 等外部密钥管理系统来增强安全性。

#### **3. Pod 如何挂载 ConfigMap / Secret**

将 ConfigMap 和 Secret 中的数据注入到 Pod 中，主要有两种方式：

**方式一：作为环境变量 (Environment Variables)**

  * **如何工作**：将 ConfigMap 或 Secret 中的一个或多个键值对，直接注入为容器的环境变量。
  * **优点**：非常简单，绝大多数应用程序都支持通过环境变量读取配置。
  * **缺点**：**非动态**。如果注入后，你更新了 ConfigMap 或 Secret，**正在运行的 Pod 的环境变量不会自动更新**。必须重启 Pod 才能加载新的配置。

**方式二：作为文件挂载 (Volume Mounts)**

  * **如何工作**：将 ConfigMap 或 Secret 作为一个虚拟的 Volume 挂载到容器内部的一个指定路径下。ConfigMap/Secret 中的每个键值对，都会成为该路径下的一个文件（文件名是 `key`，文件内容是 `value`）。
  * **优点**：**支持动态更新**！当你更新了 ConfigMap 或 Secret 后，Kubernetes 会在短时间内**自动更新**挂载到 Pod 中的对应文件内容。你的应用程序只需定期重读这个文件，就可以实现配置的动态热加载，无需重启 Pod。
  * **缺点**：需要你的应用程序支持从文件中读取配置。

-----

### **实战任务**

现在，让我们动手实践这两种方式。

#### **任务 1 & 2：创建一个 ConfigMap 并通过环境变量使用**

**场景**：我们希望为应用设置一个运行环境的标识 `APP_ENV`。

1.  **创建 ConfigMap**：
    我们可以直接用 `kubectl` 命令来创建一个简单的 ConfigMap，这比写 YAML 更快捷。

    `kubectl create configmap app-config --from-literal=APP_ENV=dev`

      * `app-config`：是我们给这个 ConfigMap起的名字。
      * `--from-literal=KEY=VALUE`：直接从字面值创建一个键值对。

    你可以用 `kubectl describe configmap app-config` 查看它的内容。

2.  **创建 Pod 来读取该变量**：
    新建 `configmap-pod.yaml` 文件。

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: configmap-env-pod
    spec:
      containers:
      - name: my-app
        image: busybox:1.35
        # 持续打印环境变量 APP_ENV 的值
        command: [ "sh", "-c", "while true; do echo Running in environment: $APP_ENV; sleep 5; done" ]
        env:
          # 定义一个环境变量
          - name: APP_ENV
            valueFrom: # 值来源于...
              configMapKeyRef: # 一个对 ConfigMap key 的引用
                name: app-config # ConfigMap 的名字
                key: APP_ENV    # ConfigMap 中的 key
      restartPolicy: Never # 方便我们查看日志后 Pod 自动退出
    ```

    **YAML 解读**：`env` 字段定义了容器的环境变量。我们创建了一个名为 `APP_ENV` 的变量，它的值 (`valueFrom`) 来源于名为 `app-config` 的 ConfigMap 中 `key` 为 `APP_ENV` 的那一项。

3.  **部署并验证**：
    `kubectl apply -f configmap-pod.yaml`
    等待 Pod 运行完成 (由于 `restartPolicy: Never` 和 `while true`，它会一直运行，我们可以手动删除它)。
    查看日志：
    `kubectl logs configmap-env-pod`

    你应该会看到输出：

    ```
    Running in environment: dev
    ```

    成功！Pod 成功地从 ConfigMap 读取到了配置。

-----

#### **任务 3 & 4：创建一个 Secret 并在 Pod 中以两种方式使用**

**场景**：我们需要将一个敏感的数据库密码 `my-super-secret-password` 安全地注入到应用 Pod 中。

1.  **创建 Secret**：
    和 ConfigMap 类似，我们用 `kubectl` 命令创建。

    `kubectl create secret generic db-credentials --from-literal=password=my-super-secret-password`

      * `generic`：表示这是一个通用的、由用户自定义的 Secret。
      * Secret 的名字是 `db-credentials`。
      * 它包含一个键值对 `password=my-super-secret-password`。

    执行 `kubectl get secret db-credentials -o yaml`，你会看到 `password` 的值是一串 Base64 编码的字符串，而不是明文。

2.  **创建 Pod 同时使用两种方式**：
    新建 `secret-pod.yaml` 文件。

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-usage-pod
    spec:
      containers:
      - name: my-app
        image: busybox:1.35
        # 这个命令会同时打印环境变量和文件内容
        command: ["sh", "-c", "echo 'Password from ENV:' $DB_PASSWORD && echo '----' && echo 'Password from file:' && cat /etc/secrets/password && sleep 3600"]
        
        # --- 方式一：注入为环境变量 ---
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: # 引用一个 Secret key
              name: db-credentials # Secret 的名字
              key: password     # Secret 中的 key
              
        # --- 方式二：挂载为文件 ---
        volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secrets" # 挂载到容器的这个路径
          readOnly: true
          
      # 在 Pod 层面定义 Volume
      volumes:
      - name: secret-volume
        secret: # Volume 的类型是 secret
          secretName: db-credentials # 引用名为 db-credentials 的 Secret
    ```

3.  **部署并验证**：
    `kubectl apply -f secret-pod.yaml`
    等待 Pod 启动，然后查看它的日志：
    `kubectl logs secret-usage-pod`

    你会看到非常清晰的输出，证明两种方式都成功了：

    ```
    Password from ENV: my-super-secret-password
    ----
    Password from file:
    my-super-secret-password
    ```

-----

### **清理环境**

完成实验后，记得删除我们创建的所有资源：

```bash
kubectl delete pod configmap-env-pod
kubectl delete configmap app-config
kubectl delete pod secret-usage-pod
kubectl delete secret db-credentials
```

-----

### **本日总结**

今天，你掌握了 Kubernetes 中配置管理的标准实践：

1.  **理解了 `ConfigMap` 和 `Secret` 的区别和用途**：前者用于通用配置，后者用于敏感数据。
2.  **学会了将配置作为环境变量注入到 Pod 中**：这是一种简单直接的方式，但配置是静态的。
3.  **学会了将配置作为 Volume 文件挂载到 Pod 中**：这种方式更强大，支持配置的动态热更新。

通过将配置与应用解耦，你的部署变得更加灵活、安全和标准化，向着云原生 "十二要素应用" 的原则又迈进了一大步。

我们已经接触到了 `emptyDir` 和 `configMap/secret` 类型的 Volume。明天，我们将深入探讨 Kubernetes 的存储核心：**Volumes**，学习如何为你的 Pod 提供真正**持久化**的存储。