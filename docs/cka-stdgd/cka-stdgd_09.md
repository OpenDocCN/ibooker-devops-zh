# 第九章：复习问题的答案

# 第二章，“集群架构、安装和配置”

1.  首先创建名为`apps`的命名空间。然后，我们将创建 ServiceAccount：

    ```
    $ kubectl create namespace apps
    $ kubectl create serviceaccount api-access -n apps
    ```

    或者，您可以使用声明性方法。从文件`apps-namespace.yaml`中的定义创建命名空间：

    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      name: apps
    ```

    从 YAML 文件创建命名空间：

    ```
    $ kubectl create -f apps-namespace.yaml
    ```

    使用以下内容创建名为`api-serviceaccount.yaml`的新 YAML 文件：

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: api-access
      namespace: apps
    ```

    运行`create`命令从 YAML 文件实例化 ServiceAccount：

    ```
    $ kubectl create -f api-serviceaccount.yaml
    ```

1.  使用`create clusterrole`命令以命令方式创建 ClusterRole：

    ```
    $ kubectl create clusterrole api-clusterrole --verb=watch,list,get \
      --resource=pods
    ```

    如果您更喜欢从 YAML 文件开始，请使用文件`api-clusterrole.yaml`中显示的内容：

    ```
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: api-clusterrole
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["watch","list","get"]
    ```

    从 YAML 文件创建 ClusterRole：

    ```
    $ kubectl create -f api-clusterrole.yaml
    ```

    使用`create clusterrolebinding`命令以命令方式创建 ClusterRoleBinding。

    ```
    $ kubectl create clusterrolebinding api-clusterrolebinding \
      --serviceaccount=apps:api-access --clusterrole=api-clusterrole
    ```

    ClusterRoleBinding 的声明性方法可能类似于文件`api-clusterrolebinding.yaml`中的方法：

    ```
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: api-clusterrolebinding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: api-clusterrole
    subjects:
    - apiGroup: ""
      kind: ServiceAccount
      name: api-access
      namespace: apps
    ```

    从 YAML 文件创建 ClusterRoleBinding：

    ```
    $ kubectl create -f api-clusterrolebinding.yaml
    ```

1.  执行`run`命令在不同的命名空间中创建 Pods。您需要在实例化 Pod`disposable`之前创建`rm`命名空间：

    ```
    $ kubectl run operator --image=nginx:1.21.1 --restart=Never \
      --port=80 --serviceaccount=api-access -n apps
    $ kubectl create namespace rm
    $ kubectl run disposable --image=nginx:1.21.1 --restart=Never \
      -n rm
    ```

    以下 YAML 清单显示了存储在文件`rm-namespace.yaml`中的`rm`命名空间定义：

    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      name: rm
    ```

    存储在文件`api-pods.yaml`中的这些 Pod 的 YAML 表示如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: operator
      namespace: apps
    spec:
      serviceAccountName: api-access
      containers:
      - name: operator
        image: nginx:1.21.1
        ports:
        - containerPort: 80
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: disposable
      namespace: rm
    spec:
      containers:
      - name: disposable
        image: nginx:1.21.1
    ```

    从 YAML 文件创建命名空间和 Pods：

    ```
    $ kubectl create -f rm-namespace.yaml
    $ kubectl create -f api-pods.yaml
    ```

1.  确定 ServiceAccount 的 API 服务器端点和 Secret 访问令牌。您将需要这些信息来进行 API 调用：

    ```
    $ kubectl config view --minify -o \
      jsonpath='{.clusters[0].cluster.server}'
    https://192.168.64.4:8443
    $ kubectl get secret $(kubectl get serviceaccount api-access -n apps \
      -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' -n apps \
      | base64 --decode
    eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1hOUhI...

    ```

    打开名为`operator`的 Pod 的交互式 Shell：

    ```
    $ kubectl exec operator -it -n apps -- /bin/sh
    ```

    发出 API 调用以列出所有在`rm`命名空间中生存的 Pods 并删除 Pod`disposable`。您会发现虽然`list`操作被允许，但`delete`操作不被允许：

    ```
    # curl https://192.168.64.4:8443/api/v1/namespaces/rm/pods --header \
    "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1hOUhI..." \
    --insecure
    {
        "kind": "PodList",
        "apiVersion": "v1",
        ...
    }
    # curl -X DELETE https://192.168.64.4:8443/api/v1/namespaces \
    /rm/pods/disposable --header \
    "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1hOUhI..." \
    --insecure
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {

      },
      "status": "Failure",
      "message": "pods \"disposable\" is forbidden: User \
      \"system:serviceaccount:apps:api-access\" cannot delete \
      resource \"pods\" in
      API group \"\" in the namespace \"rm\"",
      "reason": "Forbidden",
      "details": {
        "name": "disposable",
        "kind": "pods"
      },
      "code": 403
    }
    ```

1.  解决此示例练习需要许多手动步骤。以下命令不会输出它们的结果。

    使用 Vagrant 打开控制平面节点的交互式 Shell：

    ```
    $ vagrant ssh k8s-control-plane
    ```

    升级`kubeadm`到版本 1.21.2 并应用它：

    ```
    $ sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get \
      install -y kubeadm=1.21.2-00 && sudo apt-mark hold kubeadm
    $ sudo kubeadm upgrade apply v1.21.2
    ```

    驱逐节点，升级 kubelet 和`kubectl`，重新启动 kubelet，并取消节点的禁用状态：

    ```
    $ kubectl drain k8s-control-plane --ignore-daemonsets
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubelet=1.21.2-00 kubectl=1.21.2-00
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart kubelet
    $ kubectl uncordon k8s-control-plane
    ```

    节点的版本现在应该显示为 v1.21.2。退出节点：

    ```
    $ kubectl get nodes
    $ exit
    ```

    使用 Vagrant 打开第一个工作节点的交互式 Shell。对其他工作节点重复所有以下步骤：

    ```
    $ vagrant ssh worker-1
    ```

    升级`kubeadm`到版本 1.21.2，并将其应用到节点：

    ```
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubeadm=1.21.2-00
    $ sudo kubeadm upgrade node
    ```

    驱逐节点，升级 kubelet 和`kubectl`，重新启动 kubelet，并取消节点的禁用状态：

    ```
    $ kubectl drain worker-1 --ignore-daemonsets
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubelet=1.21.2-00 kubectl=1.21.2-00
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart kubelet
    $ kubectl uncordon worker-1
    ```

    节点的版本现在应该显示为 v1.21.2。退出节点：

    ```
    $ kubectl get nodes
    $ exit
    ```

1.  解决此示例练习需要许多手动步骤。以下命令不会输出它们的结果。

    使用 Vagrant 打开控制平面节点的交互式 Shell。这并非使用`etcdctl`命令行工具安装：

    ```
    $ vagrant ssh k8s-control-plane
    ```

    通过描述它来确定 Pod`etcd-k8s-control-plane`的参数。使用正确的参数值创建快照文件：

    ```
    $ kubectl describe pod etcd-k8s-control-plane -n kube-system
    $ sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/etcd.bak
    ```

    从快照文件恢复备份。编辑 etcd YAML 清单，并为名为`etcd-data`的卷的`spec.volumes.hostPath.path`值进行更改：

    ```
    $ sudo ETCDCTL_API=3 etcdctl --data-dir=/var/bak snapshot restore \
      /opt/etcd.bak
    $ sudo vim /etc/kubernetes/manifests/etcd.yaml
    ```

    过一段时间后，Pod `etcd-k8s-control-plane`应该会恢复到“Running”状态。退出节点：

    ```
    $ kubectl get pod etcd-k8s-control-plane -n kube-system
    $ exit
    ```

# 第三章，“工作负载”

1.  首先创建名为`nginx`的部署。使用最直接的方法来实现最快的反应时间：

    ```
    $ kubectl create deployment nginx --image=nginx:1.17.0 --replicas=2
    ```

1.  `scale`命令将副本数量增加到 7。`get`命令应该呈现出具有部署名称作为其名称前缀的七个 Pod：

    ```
    $ kubectl scale deployment nginx --replicas=7
    $ kubectl get deployments,pods
    NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/nginx   7/7     7            7           93s

    NAME                         READY   STATUS    RESTARTS   AGE
    pod/nginx-844f997cc9-6tbzw   1/1     Running   0          93s
    pod/nginx-844f997cc9-8mzz2   1/1     Running   0          93s
    pod/nginx-844f997cc9-n7g8x   1/1     Running   0          10s
    pod/nginx-844f997cc9-sbrmf   1/1     Running   0          10s
    pod/nginx-844f997cc9-wtbk6   1/1     Running   0          10s
    pod/nginx-844f997cc9-xghl9   1/1     Running   0          10s
    pod/nginx-844f997cc9-zsggj   1/1     Running   0          10s
    ```

1.  目前支持定义 CPU 和内存利用率阈值的 Horizontal Pod Autoscaler 的 API 版本是`autoscaling/v2beta2`。以下的 YAML 清单在文件`hpa.yaml`中指定了自动扩展的参数：

    ```
    apiVersion: autoscaling/v2beta2
    kind: HorizontalPodAutoscaler
    metadata:
      name: nginx-hpa
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: nginx
      minReplicas: 3
      maxReplicas: 20
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 65
      - type: Resource
        resource:
          name: memory
          target:
            type: AverageValue
            averageValue: 1Gi
    ```

    使用`create`命令创建 Horizontal Pod Autoscaler：

    ```
    $ kubectl create -f hpa.yaml
    ```

1.  您可以通过使用`edit`命令手动更改镜像名称，或者使用`set image`命令作为快捷方式。使用`--record`选项记录命令作为更改原因：

    ```
    $ kubectl set image deployment nginx nginx=nginx:1.21.1 --record
    ```

    rollout 历史记录将显示两个修订版本，起始修订版本为 1，最近的更改为 2。请注意，更改原因显示了记录的命令：

    ```
    $ kubectl rollout history deployment nginx
    deployment.apps/nginx
    REVISION  CHANGE-CAUSE
    1         <none>
    2         kubectl set image deployment nginx nginx=nginx:1.21.1 \
              --record=true
    ```

    使用`rollout undo`命令回滚到先前的修订版本。修订版本 1 变为修订版本 3：

    ```
    $ kubectl rollout undo deployment nginx --to-revision=1
    $ kubectl rollout history deployment nginx
    deployment.apps/nginx
    REVISION  CHANGE-CAUSE
    2         kubectl set image deployment nginx nginx=nginx:1.21.1 \
              --record=true
    3         <none>
    ```

1.  以下的 YAML 清单显示了类型为`kubernetes.io/basic-auth`的 Secret，存储在文件`basic-auth-secret.yaml`中：

    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: basic-auth
    type: kubernetes.io/basic-auth
    stringData:
      username: super
      password: my-s8cr3t
    ```

    使用`create`命令创建 Secret：

    ```
    $ kubectl create -f basic-auth-secret.yaml
    ```

    若要将 Secret 作为卷挂载到 Pod 模板中，请使用`edit`命令编辑 Deployment 的活动对象。Deployment 的重要部分将类似于以下的 YAML 清单：

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      replicas: 7
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.17.0
            volumeMounts:
            - mountPath: /etc/secret
              name: auth-vol
              readOnly: true
          volumes:
          - name: auth-vol
            secret:
              secretName: basic-auth
    ```

# 第四章，“调度和工具”

1.  定义资源边界的 Pod 清单存储在文件`ingress-controller-pod.yaml`中，可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: ingress-controller
    spec:
      containers:
      - name: ingress-controller
        image: bitnami/nginx-ingress-controller:1.0.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "1"
          limits:
            memory: "1024Mi"
            cpu: "2.5"
    ```

1.  假设一个由三个节点组成的集群：一个控制平面节点和两个工作节点。以下的多节点集群已经通过 Minikube 设置好。更多信息请参见[设置指南](https://oreil.ly/nVhYn)：

    ```
    $ kubectl get nodes
    NAME           STATUS   ROLES                  AGE   VERSION
    minikube       Ready    control-plane,master   41d   v1.21.2
    minikube-m02   Ready    <none>                 21h   v1.21.2
    minikube-m03   Ready    <none>                 21h   v1.21.2
    ```

    在创建对象后，您可以确定哪个节点运行了 Pod。将节点名称写入文件`node.txt`：

    ```
    $ kubectl create -f ingress-controller-pod.yaml
    $ kubectl get pod ingress-controller -o yaml | grep nodeName:
      nodeName: minikube-m02
    $ echo "minikube-m02" >> node.txt
    ```

1.  导航到包含`manifests`目录的文件夹。使用递归的`apply`命令创建`manifests`目录中包含的所有对象：

    ```
    $ kubectl apply -f manifests/ -R
    configmap/logs-config created
    pod/nginx created
    ```

1.  使用编辑器修改文件`configmap.yaml`中`dir`键的值。然后使用以下命令更新 ConfigMap 的活动对象：

    ```
    $ vim manifests/configmap.yaml
    $ kubectl apply -f manifests/configmap.yaml
    configmap/logs-config configured
    ```

    使用递归的`delete`命令删除从`manifests`目录创建的所有对象：

    ```
    $ kubectl delete -f manifests/ -R
    configmap "logs-config" deleted
    pod "nginx" deleted
    ```

1.  创建文件`kustomization.yaml`。它应该定义命名空间的公共属性，并引用文件`pod.yaml`中的资源。以下的 YAML 文件显示了它的内容：

    ```
    namespace: t012
    resources:
    - pod.yaml
    ```

    运行以下`kustomize`命令以将转换后的清单渲染为控制台输出：

    ```
    $ kubectl kustomize ./
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
      namespace: t012
    spec:
      containers:
      - image: nginx:1.21.1
        name: nginx
    ```

# 第五章，“服务和网络”

1.  首先创建命名空间`external`。在命名空间内，使用命令创建部署和 Service：

    ```
    $ kubectl create namespace external
    $ kubectl create deployment nginx --image=nginx --port=80 --replicas=3 \
      -n external
    $ kubectl create service loadbalancer nginx --tcp=80:80 -n external
    ```

    如果您更愿意使用声明性方法，请参见以下 YAML 清单：

    `external-namespace.yaml`：

    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      name: external
    ```

    `external-deployment.yaml`：

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
      namespace: external
    spec:
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
            ports:
            - containerPort: 80
    ```

    `external-service.yaml`：

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      namespace: external
    spec:
      type: LoadBalancer
      selector:
        app: nginx
      ports:
      - port: 80
        targetPort: 80
    ```

    要创建所有对象，请运行`create`或`apply`命令：

    ```
    $ kubectl create -f external-namespace.yaml
    $ kubectl create -f external-deployment.yaml
    $ kubectl create -f external-service.yaml
    ```

1.  确定类型为`LoadBalancer`的 Service 的外部 IP 地址。在下面的输出中，外部 IP 地址为`10.108.34.2`。如果使用 Minikube，请记得在另一个 Shell 中运行`minikube tunnel`，以便`EXTERNAL-IP`的值得到填充：

    ```
    $ kubectl get service -n external
    NAME    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
    nginx   LoadBalancer   10.108.34.2   10.108.34.2   80:31898/TCP   36m
    ```

    以下`curl`命令通过使用外部 IP 地址和端口调用 Service：

    ```
    $ wget 10.108.34.2:80
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    ```

1.  通过将属性`spec.type`的值从`LoadBalancer`更改为`ClusterIP`来编辑实时对象：

    ```
    $ kubectl edit service nginx -n external
    ...
    spec:
      type: ClusterIP
    ...
    ```

    现在 Service 应该指示类型为`ClusterIP`。请注意，外部 IP 地址的值已经消失。此外，静态分配的端口也消失了：

    ```
    $ kubectl get service -n external
    NAME    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
    nginx   ClusterIP   10.108.34.2   <none>        80/TCP    52m
    ```

    类型为`ClusterIP`的 Service 仅从集群内部可访问。您可以从同一命名空间内的临时 Pod 中对 Service 进行调用。您可以使用集群 IP 地址（在本例中为`10.108.34.2`）或使用 Service `nginx`的 DNS 名称：

    ```
    $ kubectl run tmp --image=busybox --restart=Never -n external -it --rm \
      -- wget 10.108.34.2:80
    Connecting to 10.108.34.2:80 (10.108.34.2:80)
    saving to 'index.html'
    index.html           100% |********************************|   615  \
    0:00:00 ETA
    'index.html' saved
    pod "tmp" deleted
    $ kubectl run tmp --image=busybox --restart=Never -n external -it --rm \
      -- wget nginx:80
    Connecting to 10.108.34.2:80 (10.108.34.2:80)
    saving to 'index.html'
    index.html           100% |********************************|   615  \
    0:00:00 ETA
    'index.html' saved
    pod "tmp" deleted

    ```

1.  要以声明性方式创建入口，运行以下命令：

    ```
    $ kubectl create ingress incoming --rule="/*=nginx:80" -n external

    ```

    如果您更愿意使用声明性方法，请参见此处显示的文件`incoming-ingress.yaml`中的 YAML 清单：

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: incoming
      namespace: external
    spec:
      rules:
      - http:
          paths:
          - backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
    ```

    要创建对象，请运行`create`或`apply`命令：

    ```
    $ kubectl create -f incoming-ingress.yaml
    ```

1.  要验证 Ingress 的正确行为，请获取集群中任何节点的 IP 地址。在这里，我们只处理具有 IP 地址`192.168.64.19`的单个节点：

    ```
    $ kubectl get nodes -o wide
    NAME      STATUS   ROLES                 AGE   VERSION   INTERNAL-IP   \
    EXTERNAL-IP   OS-IMAGE               KERNEL-VERSION   CONTAINER-RUNTIME
    minikube  Ready    control-plane,master  13d   v1.21.2   192.168.64.19 \
    <none>        Buildroot 2020.02.12   4.19.182         docker://20.10.6
    ```

    Ingress 已配置为访问任何主机名的调用。从本地机器对节点的 IP 地址进行调用。流量将通过名为`nginx`的 Service 路由到 Pods：

    ```
    $ curl 192.168.64.19
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    ```

1.  为了快速实现，您可以使用`run`命令加上`--expose` CLI 选项一起创建`echoserver` Pod 和 Service：

    ```
    $ kubectl run echoserver --image=k8s.gcr.io/echoserver:1.10 \
      --restart=Never --port=8080 --expose -n external
    ```

    声明性方法需要创建以下 YAML 清单：

    `external-echoserver-pod.yaml`:

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: echoserver
      namespace: external
      labels:
        run: nginx
    spec:
      containers:
      - name: echoserver
        image: k8s.gcr.io/echoserver:1.10
        ports:
        - containerPort: 8080
    ```

    `external-echoserver-service.yaml`：

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: echoserver
      namespace: external
    spec:
      type: ClusterIP
      selector:
        run: nginx
      ports:
      - port: 8080
        targetPort: 8080
    ```

    要创建对象，请运行`create`或`apply`命令：

    ```
    $ kubectl create -f external-echoserver-pod.yaml
    $ kubectl create -f external-echoserver-service.yaml
    ```

    修改现有的 Ingress 并添加一个新的规则，将流量路由到`echoservice` Service。生成的 YAML 定义如下所示：

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: incoming
      namespace: external
    spec:
      rules:
      - http:
          paths:
          - backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
          - backend:
              service:
                name: echoserver
                port:
                  number: 8080
            path: /echo
            pathType: Exact
    ```

1.  从本地机器对节点的 IP 地址进行调用，路径为`/echo`。流量将通过名为`echoservice`的 Service 路由到 Pods：

    ```
    $ curl 192.168.64.19/echo

    Hostname: echoserver
    ...
    ```

1.  您可以通过使用命令`kubectl edit configmap coredns -n kube-system`编辑命名空间`kube-system`中的 ConfigMap `coredns`来自定义 CoreDNS 设置。以下 YAML 清单显示了`rewrite`规则：

    ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: coredns-custom
      namespace: kube-system
    data:
      Corefile: |
        .:53 {
            ...
            rewrite name substring svc.cka.example.com svc.cluster.local
            kubernetes cluster.local in-addr.arpa ip6.arpa {
            ...
        }
    ```

    在命名空间`kube-system`中查找运行 CoreDNS 的 Pod，并删除它以强制重新创建 Pod。您应该看到 Kubernetes 为 CoreDNS Pod 创建了一个新对象：

    ```
    $ kubectl get pods -n kube-system
    NAME                               READY   STATUS    RESTARTS   AGE
    coredns-558bd4d5db-kjdtx           1/1     Running   0          9m35s
    ...
    $ kubectl delete pod coredns-558bd4d5db-kjdtx -n kube-system
    $ kubectl get pods -n kube-system
    NAME                               READY   STATUS    RESTARTS   AGE
    coredns-558bd4d5db-mc98t           1/1     Running   0          54s
    ...
    ```

1.  如果尚不存在，请创建命名空间`hello`。从`hello`命名空间的临时 Pod 针对`echoserver.external.svc.cka.example.com`运行`wget`命令。调用应成功：

    ```
    $ kubectl create namespace hello
    $ kubectl run tmp --image=busybox --restart=Never -n hello -it --rm \
      -- wget echoserver.external.svc.cka.example.com:8080
    Connecting to echoserver.external.svc.cka.example.com:8080 \
    (10.104.248.24:8080)
    saving to 'index.html'
    index.html           100% |********************************|   \
    460  0:00:00 ETA
    'index.html' saved
    pod "tmp" deleted

    ```

# 第六章，“存储”

1.  首先创建名为`logs-pv.yaml`的新文件。内容如下：

    ```
    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: logs-pv
    spec:
      capacity:
        storage: 2Gi
      accessModes:
        - ReadWriteOnce
        - ReadOnlyMany
      persistentVolumeReclaimPolicy: Delete
      storageClassName: ""
      hostPath:
        path: /tmp/logs
    ```

    创建持久卷对象并检查其状态：

    ```
    $ kubectl create -f logs-pv.yaml
    persistentvolume/logs-pv created
    $ kubectl get pv
    NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS \
      CLAIM              STORAGECLASS   REASON   AGE
    logs-pv   2Gi        RWO,ROX        Delete           Bound  \
      default/logs-pvc                           16m
    ```

1.  创建文件`logs-pvc.yaml`以定义持久卷声明。以下 YAML 清单显示了其内容：

    ```
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: logs-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ""
      resources:
        requests:
          storage: 1Gi
    ```

    创建持久卷声明对象并检查其状态：

    ```
    $ kubectl create -f logs-pvc.yaml
    persistentvolumeclaim/logs-pvc created
    $ kubectl get pvc
    NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES \
      STORAGECLASS   AGE
    logs-pvc   Bound    logs-pv   2Gi        RWO,ROX      \
                     17m
    ```

1.  使用`--dry-run`命令行选项创建基本的 YAML 清单：

    ```
    $ kubectl run nginx --image=nginx --dry-run=client --restart=Never \
      -o yaml > nginx-pod.yaml
    ```

    现在，编辑文件`nginx-pod.yaml`并将持久卷声明绑定到它：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
      name: nginx
    spec:
      volumes:
        - name: logs-volume
          persistentVolumeClaim:
            claimName: logs-pvc
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
          - mountPath: "/var/log/nginx"
            name: logs-volume
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    ```

    使用以下命令创建 Pod 并检查其状态：

    ```
    $ kubectl create -f nginx-pod.yaml
    pod/nginx created
    $ kubectl get pods
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          8s
    ```

1.  使用`exec`命令打开与 Pod 交互的 shell，并在挂载的目录中创建文件：

    ```
    $ kubectl exec nginx -it -- /bin/sh
    # cd /var/log/nginx
    # touch my-nginx.log
    # ls
    access.log  error.log  my-nginx.log
    # exit
    ```

1.  删除 Pod 和持久卷声明。由于其回收策略，持久卷将被自动删除：

    ```
    $ kubectl delete pod nginx
    $ kubectl delete pvc logs-pvc
    $ kubectl get pv,pvc
    No resources found
    ```

1.  您可以使用以下命令列出存储类。如果使用 Minikube，可能只会找到一个存储类，默认的存储类名为`standard`，使用提供程序`k8s.io/minikube-hostpath`：

    ```
    $ kubectl get sc
    NAME                 PROVISIONER                RECLAIMPOLICY \
      VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    standard (default)   k8s.io/minikube-hostpath   Delete        \
      Immediate           false                  22d
    ```

1.  为存储类创建文件`custom-sc.yaml`。YAML 清单可能如下所示：

    ```
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: custom
    provisioner: k8s.io/minikube-hostpath
    ```

    使用以下命令创建存储类。列出所有存储类会显示默认存储类和新的存储类：

    ```
    $ kubectl create -f custom-sc.yaml
    storageclass.storage.k8s.io/custom created
    $ kubectl get sc
    NAME                 PROVISIONER                RECLAIMPOLICY \
      VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    custom               k8s.io/minikube-hostpath   Delete        \
      Immediate           false                  11s
    standard (default)   k8s.io/minikube-hostpath   Delete        \
      Immediate           false                  22d
    ```

1.  创建文件`custom-pvc.yaml`以定义持久卷声明。以下 YAML 清单显示了其内容：

    ```
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: custom-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: custom
      resources:
        requests:
          storage: 500Mi
    ```

    创建持久卷声明对象并检查其状态：

    ```
    $ kubectl create -f custom-pvc.yaml
    persistentvolumeclaim/custom-pvc created
    $ kubectl get pv,pvc
    NAME                                                        CAPACITY \
      ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM   STORAGECLASS \
        REASON   AGE
    persistentvolume/pvc-6fe081b5-e425-45cd-a94a-3488ce24cb87   500Mi    \
      RWO            Delete           Bound    default/custom-pvc   custom \
                       13s

    NAME                               STATUS \
      VOLUME                                     CAPACITY   ACCESS MODES \
       STORAGECLASS   AGE
    persistentvolumeclaim/custom-pvc   Bound  \
      pvc-6fe081b5-e425-45cd-a94a-3488ce24cb87   500Mi      RWO \
                custom           13s
    ```

1.  将持久卷的名称写入文件`pv-name.txt`：

    ```
    $ echo "pvc-6fe081b5-e425-45cd-a94a-3488ce24cb87" > pv-name.txt
    ```

1.  删除持久卷声明将同时删除绑定的持久卷：

    ```
    $ kubectl delete pvc custom-pvc
    $ kubectl get pv,pvc
    No resources found
    ```

# 第七章，“故障排除”

1.  首先创建名为`multi-container.yaml`的 Pod 的 YAML 起点文件。以下命令创建文件：

    ```
    $ kubectl run multi --image=nginx:1.21.6 -o yaml --dry-run=client \
      --restart=Never > multi-container.yaml
    ```

    编辑 YAML 清单。添加 sidecar 容器。以下 YAML 文件的内容可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: multi
    spec:
      containers:
      - image: nginx:1.21.6
        name: nginx
      - image: busybox:1.35.0
        name: streaming
        args: [/bin/sh, -c, 'tail -n+1 -f /var/log/nginx/access.log']
    ```

1.  添加卷定义并将其挂载到两个容器中。最终的 YAML 清单如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: multi
    spec:
      containers:
      - image: nginx:1.21.6
        name: nginx
        volumeMounts:
        - name: accesslog
          mountPath: /var/log/nginx
      - image: busybox:1.35.0
        name: streaming
        args: [/bin/sh, -c, 'tail -n+1 -f /var/log/nginx/access.log']
        volumeMounts:
        - name: accesslog
          mountPath: /var/log/nginx
      volumes:
      - name: accesslog
        emptyDir: {}
    ```

    通过指向 YAML 文件的`create`命令创建 Pod：

    ```
    $ kubectl create -f multi-container.yaml
    ```

1.  确定 Pod 的 IP 地址。在以下示例中，IP 地址为`10.244.2.3`。使用临时 Pod 对 nginx 进行三次调用。`streaming`容器的日志有三个条目：

    ```
    $ kubectl get pod multi -o wide
    NAME    READY   STATUS    RESTARTS   AGE     IP           NODE         \
      NOMINATED NODE   READINESS GATES
    multi   2/2     Running   0          3m23s   10.244.2.3   minikube-m03 \
      <none>           <none>
    $ kubectl run tmp --image=busybox --restart=Never -it --rm \
      -- wget 10.244.2.3
    $ kubectl run tmp --image=busybox --restart=Never -it --rm \
      -- wget 10.244.2.3
    $ kubectl run tmp --image=busybox --restart=Never -it --rm \
      -- wget 10.244.2.3
    $ kubectl logs multi -c streaming
    10.244.1.2 - - [27/Jan/2022:16:44:25 +0000] "GET / HTTP/1.1" 200 \
    615 "-" "Wget" "-"
    10.244.1.3 - - [27/Jan/2022:16:44:29 +0000] "GET / HTTP/1.1" 200 \
    615 "-" "Wget" "-"
    10.244.1.4 - - [27/Jan/2022:16:44:32 +0000] "GET / HTTP/1.1" 200 \
    615 "-" "Wget" "-"
    ```

1.  创建 Pods `stress-1`和`stress-2`。以下 YAML 清单显示了名为`stress-1`的 Pod 在文件`stress-1-pod.yaml`中的定义。创建第二个 YAML 文件，并相应更改 Pod 名称：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: stress-1
    spec:
      containers:
      - image: polinux/stress:1.0.4
        name: consumer
        resources:
          limits:
            memory: "250Mi"
          requests:
            memory: "250Mi"
        args: [/bin/sh, -c, 'stress --vm 1 --vm-bytes \
               $(shuf -i 20-200 -n 1)M --vm-hang 1']
    ```

    创建 Pod 并检查其状态：

    ```
    $ kubectl create -f stress-1-pod.yaml
    $ kubectl create -f stress-2-pod.yaml
    $ kubectl get pods
    NAME       READY   STATUS    RESTARTS   AGE
    stress-1   1/1     Running   0          15m
    stress-2   1/1     Running   0          6m28s
    ```

1.  如果集群尚未安装[metrics server](https://oreil.ly/e1NSC)，请安装它。从 metrics 服务器检索 Pod 的指标。在下面的示例中，名为`stress-2`的 Pod 消耗了更多内存。将 Pod 名称写入文件*max-memory.txt*：

    ```
    $ kubectl top pods
    NAME       CPU(cores)   MEMORY(bytes)
    stress-1   32m          93Mi
    stress-2   47m          117Mi
    $ echo "stress-2" > max-memory.txt
    ```

1.  你可以在检出的 GitHub 仓库 [*bmuschko/cka-study-guide*](https://oreil.ly/jUIq8) 的文件 [*app-a/ch07/troubleshooting-pod/solution/solution.md*](https://oreil.ly/BjXvd) 中找到解决方案。

1.  你可以在检出的 GitHub 仓库 [*bmuschko/cka-study-guide*](https://oreil.ly/jUIq8) 的文件 [*app-a/ch07/troubleshooting-deployment/solution/solution.md*](https://oreil.ly/PQEQt) 中找到解决方案。

1.  你可以在检出的 GitHub 仓库 [*bmuschko/cka-study-guide*](https://oreil.ly/jUIq8) 的文件 [*app-a/ch07/troubleshooting-service/solution/solution.md*](https://oreil.ly/oPmYR) 中找到解决方案。

1.  你可以在检出的 GitHub 仓库 [*bmuschko/cka-study-guide*](https://oreil.ly/jUIq8) 的文件 [*app-a/ch07/troubleshooting-control-plane-node/solution/solution.md*](https://oreil.ly/CcLJe) 中找到解决方案。

1.  你可以在检出的 GitHub 仓库 [*bmuschko/cka-study-guide*](https://oreil.ly/jUIq8) 的文件 [*app-a/ch07/troubleshooting-worker-node/solution/solution.md*](https://oreil.ly/WI3sa) 中找到解决方案。
