Of course\! Welcome to your Day 17 learning plan. Today, we shift our focus from bringing traffic *in* to controlling traffic *within* the cluster. This is a critical aspect of security and multi-tenancy in Kubernetes.

Let's get started on mastering NetworkPolicy.

-----

### 🎯 学习内容：理论知识篇

Before we start locking down our network, let's understand the principles behind Kubernetes networking security.

#### 1\. 默认情况：Pod 间网络是全通的 (Default Open Network)

In a standard Kubernetes installation, the network model is intentionally flat and open. This means:
**By default, any Pod can communicate with any other Pod in the cluster, regardless of which Namespace they are in.**

This is designed for simplicity and ease of service discovery. However, in a production environment, especially one with multiple applications or teams, this "trust everyone" model is a significant security risk. A compromised container in one application could potentially attack services in a completely different application.

This is precisely the problem that `NetworkPolicy` solves.

#### 2\. NetworkPolicy：集群内部的防火墙

`NetworkPolicy` is a Kubernetes resource that acts like a virtual firewall for your Pods. It allows you to specify how groups of Pods are allowed to communicate with each other and with other network endpoints.

Key concepts to understand about NetworkPolicy:

  * **Opt-in Model**: Network policies are "opt-in". This means that until you apply a policy that selects a specific Pod, it is not restricted and will continue to accept traffic from all sources (the default open behavior).
  * **Selector-based**: Policies don't apply to Pods directly by name. Instead, they use **labels** to select groups of Pods. This makes them incredibly flexible and scalable.
  * **Default Deny Principle**: **Once a Pod is selected by any `NetworkPolicy`, it becomes isolated.** It will reject any connection that is not explicitly allowed by a policy. This "default deny" stance is the foundation of network security.
  * **Directional Rules**: Policies can specify rules for `ingress` (incoming traffic) and `egress` (outgoing traffic).
      * **Ingress**: Controls traffic coming *to* the selected Pods.
      * **Egress**: Controls traffic going *from* the selected Pods.

A typical `NetworkPolicy` YAML has three main sections:

1.  `podSelector`: Selects the Pods to which this policy applies. If left empty, it applies to all Pods in the namespace.
2.  `policyTypes`: Specifies whether the policy includes `Ingress` rules, `Egress` rules, or both.
3.  `ingress` / `egress`: Defines the actual allow rules for incoming or outgoing traffic, specifying sources (`from`) or destinations (`to`) based on Pod selectors, namespace selectors, or IP blocks.

#### 3\. 常见应用场景：只允许前端访问后端

This is the classic "three-tier application" security model.

  * A `backend` application (e.g., a database or API) should only accept connections from the `frontend` application.
  * It should reject connections from any other Pods in the cluster to prevent unauthorized access.

To achieve this, we would create a `NetworkPolicy` that:

1.  Selects the `backend` Pods using a label (e.g., `app: backend`).
2.  Defines an `ingress` rule.
3.  In that rule, explicitly allows traffic `from` Pods that have the `frontend` label (e.g., `app: frontend`).

Once this policy is applied, the `backend` Pod will only accept connections from the `frontend` Pod. All other connections, including from other Pods in the same namespace or even from the Kubernetes nodes themselves, will be blocked.

Theory complete. Time to put it into practice.

-----

### 🎯 实战任务：动手操作篇

Let's build and enforce this frontend-backend security model.

#### ⚠️ **重要前提：检查你的网络插件**

NetworkPolicy functionality is not handled by Kubernetes itself, but by the **network plugin (CNI)** you are using. Some simple CNIs do not enforce NetworkPolicy. The most popular ones like **Calico, Cilium, and Weave Net** do.

Minikube's default driver might not support it. To ensure your experiment works, it's best to start minikube with a compatible CNI. Calico is an excellent choice.

If your minikube is already running, please stop and delete it (`minikube stop && minikube delete`), then restart it with this command:

```bash
# This command starts minikube and installs the Calico CNI, which enforces Network Policies.
minikube start --network-plugin=cni --cni=calico
```

This might take a bit longer to start as it has to set up Calico. It is essential for this lesson.

#### 第 1 步：部署 Frontend 和 Backend

We need three components for our test: a backend, a frontend, and another "untrusted" pod to test from.

Let's create a YAML file named `app-for-policy.yaml` with the following content. Pay close attention to the `labels`.

```yaml
# app-for-policy.yaml

# ----------------- Backend App (NGINX) ----------------- #
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        # The label that our NetworkPolicy will select
        app: backend
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.21
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    # This service targets pods with the 'app: backend' label
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

# ----------------- Frontend App (Busybox) ----------------- #
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        # The label that our NetworkPolicy will use in the 'from' rule
        app: frontend
    spec:
      containers:
      - name: busybox-container
        image: busybox:1.28
        # Keep the container running with a sleep command
        command: ['sh', '-c', 'sleep 3600']
```

Now, apply this manifest:

```bash
kubectl apply -f app-for-policy.yaml
```

**验证部署:**

```bash
kubectl get pods -l app --show-labels
```

You should see one `backend` pod and one `frontend` pod running, with their respective labels.

#### 第 2 步：验证默认的全通网络

Before applying the policy, let's confirm that anyone can access the backend.

1.  **从 `frontend` Pod 访问 `backend` (应该成功):**

    ```bash
    # Get the name of the frontend pod
    FRONTEND_POD=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')

    # Execute a curl command from within the frontend pod to the backend service
    kubectl exec $FRONTEND_POD -- curl -s --connect-timeout 2 backend-service
    ```

    You should see the "Welcome to nginx\!" HTML page. This is expected.

2.  **从一个 "untrusted" Pod 访问 `backend` (也应该成功):**
    Let's simulate another random pod trying to access our backend.

    ```bash
    # Run a temporary busybox pod and try to curl the backend
    kubectl run --rm -it untrusted-pod --image=busybox:1.28 -- sh -c "curl -s --connect-timeout 2 backend-service"
    ```

    Again, you will see the Nginx welcome page. This proves our network is currently wide open.

#### 第 3 步：创建并应用 NetworkPolicy

Now, let's lock it down. Create a file named `backend-policy.yaml`.

```yaml
# backend-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  # 1. Select the pods this policy applies to
  podSelector:
    matchLabels:
      app: backend
  
  # 2. Define the types of rules included in this policy
  policyTypes:
  - Ingress

  # 3. List the actual ingress (incoming) rules
  ingress:
  - from:
    # Allow traffic from pods with this label
    - podSelector:
        matchLabels:
          app: frontend
    # You could also specify ports to allow traffic only to a specific port
    ports:
    - protocol: TCP
      port: 80
```

This policy reads: "For any Pod with the label `app: backend`, only allow incoming TCP traffic on port 80 from Pods that have the label `app: frontend`."

Apply the policy:

```bash
kubectl apply -f backend-policy.yaml
```

**验证策略:**

```bash
kubectl get networkpolicy
```

#### 第 4 步：验证访问已被正确控制

Now we repeat the tests from Step 2 and observe the new behavior.

1.  **从 `frontend` Pod 再次访问 `backend` (应该仍然成功):**

    ```bash
    # Get the name of the frontend pod again (in case it changed)
    FRONTEND_POD=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')

    # Execute the curl command again
    kubectl exec $FRONTEND_POD -- curl -s --connect-timeout 2 backend-service
    ```

    You should **still see the Nginx welcome page**. Our policy is working as intended, allowing this specific connection.

2.  **从 "untrusted" Pod 再次访问 `backend` (现在应该失败):**

    ```bash
    # Run the temporary pod again
    kubectl run --rm -it untrusted-pod --image=busybox:1.28 -- sh -c "curl -s --connect-timeout 2 backend-service"
    ```

    **This command will now hang for 2 seconds and then time out\!** You will see an error message like `curl: (28) Connection timed out after 2001 milliseconds`. This is the proof that our `NetworkPolicy` has successfully isolated the backend pod and is blocking unauthorized traffic.

Congratulations\! You have successfully implemented a network security policy in Kubernetes.

-----

### 🧹 清理环境

Let's clean up the resources we created today.

1.  **删除 NetworkPolicy:**
    ```bash
    kubectl delete -f backend-policy.yaml
    ```
2.  **删除应用和 Service:**
    ```bash
    kubectl delete -f app-for-policy.yaml
    ```
3.  **停止或删除 Minikube 集群:**
    ```bash
    minikube stop
    # or
    # minikube delete
    ```

-----

### 总结

Today, on Day 17, you've taken a huge step in securing your Kubernetes workloads:

1.  **Understood the Default**: You learned that Kubernetes networking is wide open by default, which is a security concern.
2.  **Mastered the Concept**: You now understand that `NetworkPolicy` acts as a firewall for Pods, using an "opt-in, default-deny" model based on labels.
3.  **Implemented a Real-World Scenario**: You successfully created a policy to isolate a backend service, allowing access only from a designated frontend, and verified that it blocks all other traffic.

This skill is fundamental for building secure, robust, and multi-tenant applications on Kubernetes. Fantastic work today\!