部署 Pod：在终端中，进入 nginx-pod.yaml 文件所在的目录，执行以下命令：
kubectl apply -f nginx-pod.yaml

检查 Pod 状态：执行命令查看 Pod 是否正在运行：
kubectl get pods

执行 describe 命令：
kubectl describe pod my-nginx-pod

部署 Pod：
kubectl apply -f multi-container-pod.yaml

检查 Pod 状态：等待 Pod 变为 Running 状态，并且 READY 状态显示为 2/2，这表示两个容器都已准备就绪。
kubectl get pods

执行以下命令，进入 busybox 容器的 shell：
kubectl exec -it multi-container-demo -c busybox-sidecar -- sh

在 busybox 内部访问 nginx：因为它们在同一个 Pod 里，共享网络，所以 busybox-sidecar 可以通过localhost 直接访问到 nginx-main 容器的 80 端口。在 busybox 的 shell 中执行 wget 命令：
wget -qO- localhost:80

删除我们创建的 Pod，以释放资源。
kubectl delete pod my-nginx-pod
kubectl delete pod multi-container-demo