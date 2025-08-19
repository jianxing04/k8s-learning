部署 Deployment：
kubectl apply -f nginx-deployment.yaml

检查状态：
你可以同时查看 Deployment, ReplicaSet 和 Pod 的状态。
kubectl get deployment
kubectl get rs (rs 是 ReplicaSet 的缩写)
kubectl get pods

修改配置：
kubectl set image deployment/my-nginx-deployment nginx=nginx:1.25

立即观察！
马上打开另一个终端窗口，快速执行下面的命令，加上 --watch 参数来实时监控化：
kubectl get pods --watch

检查更新后的 ReplicaSet： 
kubectl get rs

用 kubectl scale 把副本数改成 5
kubectl scale deployment my-nginx-deployment --replicas=5

检查 Pod 数量：
kubectl get pods

查看历史版本：
kubectl rollout history deployment my-nginx-deployment

执行回滚命令：这个命令会回滚到上一个版本（也就是 REVISION 1）。
kubectl rollout undo deployment my-nginx-deployment

再次观察：再次使用 
kubectl get pods --watch

验证回滚结果：回滚完成后，检查 ReplicaSet：
kubectl get rs

查看镜像版本是否已经变回 nginx:1.24
kubectl describe deployment my-nginx-deployment | grep Image

清理环境,删除今天的 Deployment（它会自动删除关联的 ReplicaSet 和 Pod）：
kubectl delete deployment my-nginx-deployment