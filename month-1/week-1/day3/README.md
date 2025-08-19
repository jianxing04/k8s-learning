执行部署命令：
kubectl apply -f nginx-deployment.yaml

确认 3 个 nginx Pod 正在运行：
kubectl get pods

应用 Service 配置：
kubectl apply -f nginx-clusterip-service.yaml

查看 Service：
kubectl get service 或 kubectl get svc

启动一个临时的 busybox Pod：
我们将使用 kubectl run 命令来快速创建一个临时的、可交互的 Pod。
kubectl run tmp-client --rm -it --image=busybox -- sh

从 tmp-client 内部访问 Service：
用 wget 或 curl 访问我们刚刚创建的 Service 的 ClusterIP（请替换成你自己的 ClusterIP）。
wget -qO- 10.108.111.22

通过 Service 名称访问（更推荐的方式）：
在 Pod 内，直接使用 Service 的名字作为域名来访问，更为方便。
wget -qO- my-nginx-clusterip

应用配置：
kubectl apply -f nginx-nodeport-service.yaml

查看 Service 并找到 NodePort：
kubectl get svc my-nginx-nodeport

从宿主机访问：
方法一 (Minikube 快捷方式)：如果使用 Minikube，它提供了一个非常方便的命令。
minikube service my-nginx-nodeport

方法二 (通用方式)：
首先，获取 Minikube 虚拟机（也就是我们的 Kubernetes Node）的 IP 地址。
minikube ip

假设得到的 IP 是 192.168.49.2。然后，使用这个 IP 加上我们刚才查到的 NodePort 31978 来访问。在你的电脑终端里执行：
curl http://$(minikube ip):31978

或者直接在浏览器地址栏输入 
http://192.168.49.2:31978

删除今天创建的所有资源：
kubectl delete deployment my-nginx-deployment
kubectl delete service my-nginx-clusterip
kubectl delete service my-nginx-nodeport