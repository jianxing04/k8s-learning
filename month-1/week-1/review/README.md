好 👌 那我给你写一个 **完整实验步骤**，让你从 0 到 1 演练一遍这个复习项目。
这样你既能复习命令，也能把 Deployment/Service/Ingress/滚动更新/回退 都过一遍。

---

# 🔬 K8s 复习实验步骤

## 0. 准备工作

1. 确认你已经安装好 **kubectl + minikube**，并且 ingress 插件已启用：

   ```bash
   minikube addons enable ingress
   ```
2. 在 `/etc/hosts` 文件里添加一行，保证 `mall.local` 能解析到你的 Minikube 节点：

   ```bash
   <minikube_ip> mall.local
   ```

   > 查看 minikube IP:
   >
   > ```bash
   > minikube ip
   > ```

---

## 1. 部署项目

1. 创建 `mall-demo.yaml`，复制我之前给你的 YAML 文件。
2. 应用：

   ```bash
   kubectl apply -f mall-demo.yaml
   ```

---

## 2. 验证 Deployment 和 Pod

1. 查看 Deployment：

   ```bash
   kubectl get deployments
   ```

   你应该能看到 `user-deployment` 和 `product-deployment`。
2. 查看 Pod：

   ```bash
   kubectl get pods -o wide
   ```

   * 每个 Deployment 启动 2 个 Pod。
   * 注意 Pod 启动时会先运行 **initContainer**，你可以用：

     ```bash
     kubectl describe pod <某个pod名>
     ```

     在 `Init Containers` 部分看到 init 的执行记录。

---

## 3. 验证 Service

1. 查看 Service：

   ```bash
   kubectl get svc
   ```

   你会看到 `user-service` 和 `product-service`，类型是 `ClusterIP`。
2. 在集群内测试 Service：

   ```bash
   kubectl run curl --image=radial/busyboxplus:curl -it --rm -- \
     curl http://user-service
   kubectl run curl --image=radial/busyboxplus:curl -it --rm -- \
     curl http://product-service
   ```

   这会返回 nginx / httpd 的默认页面。

---

## 4. 验证 Ingress

1. 查看 Ingress：

   ```bash
   kubectl get ingress
   ```

   你会看到 `mall-ingress`。
2. 从宿主机测试：

   ```bash
   curl -H "Host: mall.local" http://$(minikube ip)/user
   curl -H "Host: mall.local" http://$(minikube ip)/product
   ```

   * `/user` → nginx 页面
   * `/product` → httpd 页面

---

## 5. 滚动更新实验

1. 更新 Deployment 的镜像：

   ```bash
   kubectl set image deployment/user-deployment user-app=nginx:1.25
   ```

   这会触发滚动更新。
2. 观察滚动更新过程：

   ```bash
   kubectl rollout status deployment/user-deployment
   ```

   你会看到 Pod 逐个替换（新 Pod 启动，旧 Pod 关闭）。
3. 再次访问：

   ```bash
   curl -H "Host: mall.local" http://$(minikube ip)/user
   ```

   页面依然正常说明滚动更新完成。

---

## 6. 回退实验

1. 假设你更新到错误镜像：

   ```bash
   kubectl set image deployment/user-deployment user-app=nginx:doesnotexist
   ```

   你会看到 Pod 无法拉取镜像。
2. 回退到上一个版本：

   ```bash
   kubectl rollout undo deployment/user-deployment
   ```

   或者回退到指定版本：

   ```bash
   kubectl rollout undo deployment/user-deployment --to-revision=2
   ```
3. 验证 Deployment 恢复正常：

   ```bash
   kubectl get pods
   ```

---

# ✅ 实验复习点

* **Deployment**：声明式管理 Pod，支持滚动更新与回退。
* **initContainer**：Pod 启动前的准备任务。
* **Service**：集群内访问的稳定入口。
* **Ingress**：集群外访问入口，基于路径转发。
* **滚动更新与回退**：`kubectl set image` + `kubectl rollout undo`。

---

要不要我帮你再整理一份 **命令速查表**（常用的 kubectl 命令，方便你考试/实战时快速回忆）？


好嘞 👍 我来给你整理一份 **Kubernetes 命令速查表**，专门针对你已经学过的内容（Deployment、Service、Ingress、initContainer、滚动更新、回退等）。

---

# 🚀 K8s 命令速查表

## 1. 基础操作

```bash
# 查看集群信息
kubectl cluster-info

# 查看所有命名空间
kubectl get ns

# 查看所有资源（Pod, Service, Deployment, Ingress）
kubectl get all
```

---

## 2. Pod 相关

```bash
# 查看 Pod
kubectl get pods

# 查看 Pod 详细信息（包括 initContainer）
kubectl describe pod <pod-name>

# 查看 Pod 日志
kubectl logs <pod-name>

# 如果 Pod 有多个容器，指定容器名
kubectl logs <pod-name> -c <container-name>

# 进入 Pod 内部（调试用）
kubectl exec -it <pod-name> -- /bin/sh
```

---

## 3. Deployment 相关

```bash
# 查看 Deployment
kubectl get deployments

# 查看 Deployment 详细信息
kubectl describe deployment <deployment-name>

# 修改 Deployment 镜像（触发滚动更新）
kubectl set image deployment/<deployment-name> <container-name>=<new-image>

# 查看滚动更新状态
kubectl rollout status deployment/<deployment-name>

# 查看 Deployment 的历史版本
kubectl rollout history deployment/<deployment-name>

# 回退到上一个版本
kubectl rollout undo deployment/<deployment-name>

# 回退到指定版本
kubectl rollout undo deployment/<deployment-name> --to-revision=<revision>
```

---

## 4. Service 相关

```bash
# 查看 Service
kubectl get svc

# 查看 Service 详细信息
kubectl describe svc <service-name>

# 在集群内测试 Service
kubectl run curl --image=radial/busyboxplus:curl -it --rm -- \
  curl http://<service-name>:<port>
```

---

## 5. Ingress 相关

```bash
# 查看 Ingress
kubectl get ingress

# 查看 Ingress 详细信息
kubectl describe ingress <ingress-name>

# 从宿主机测试 Ingress
curl -H "Host: mall.local" http://$(minikube ip)/user
```

---

## 6. 资源管理

```bash
# 应用 YAML 文件
kubectl apply -f <file>.yaml

# 删除资源
kubectl delete -f <file>.yaml

# 按类型删除
kubectl delete deployment <name>
kubectl delete svc <name>
kubectl delete ingress <name>

# 编辑资源（会打开编辑器）
kubectl edit deployment <name>
```

---

## 7. 调试与排错

```bash
# 查看事件（常用于 Pod 启动失败时）
kubectl get events

# 查看 Pod 启动过程和失败原因
kubectl describe pod <pod-name>

# 跟踪 Pod 日志（类似 tail -f）
kubectl logs -f <pod-name>
```

---

# ✅ 总结

* **Pod** → `kubectl get pods` / `kubectl logs` / `kubectl exec`
* **Deployment** → `kubectl set image` / `kubectl rollout`（更新 & 回退）
* **Service** → `kubectl get svc` / `curl` 内部访问
* **Ingress** → `kubectl get ingress` / `curl -H "Host:..."` 外部访问

---

要不要我帮你把这份 **速查表做成 HTML 文件**，这样你就能放到本地随时打开复习？
