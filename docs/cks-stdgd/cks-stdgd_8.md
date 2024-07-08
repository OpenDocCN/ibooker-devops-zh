# 附录：回答审查问题

# 第二章，“集群设置”

1.  创建名为 `deny-egress-external.yaml` 的文件以定义网络策略。网络策略需要将 Pod 选择器设置为 `app=backend` 并定义 `Egress` 策略类型。确保允许协议 UDP 和 TCP 的端口 53。出站策略的命名空间选择器需要使用 `{}` 来选择所有命名空间：

    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-egress-external
    spec:
      podSelector:
        matchLabels:
          app: backend
      policyTypes:
      - Egress
      egress:
      - to:
        - namespaceSelector: {}
        ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    ```

    运行 `apply` 命令从 YAML 文件实例化网络策略对象：

    ```
    $ kubectl apply -f deny-egress-external.yaml
    ```

1.  符合网络策略的标签选择的 Pod 可以调用集群外的 URL。在这种情况下，标签分配为 `app=frontend`：

    ```
    $ kubectl run web --image=busybox:1.36.0 -l app=frontend --port=80 -it \
      --rm --restart=Never -- wget http://google.com --timeout=5 --tries=1
    Connecting to google.com (142.250.69.238:80)
    Connecting to www.google.com (142.250.72.4:80)
    saving to /'index.xhtml'
    index.xhtml           100% |************| 13987 \
    0:00:00 ETA
    /'index.xhtml' saved
    pod "web" deleted**********
    ```

*****   符合网络策略的标签选择的 Pod 不能调用集群外的 URL。在这种情况下，标签分配为 `app=backend`：

    ```
    $ kubectl run web --image=busybox:1.36.0 -l app=backend --port=80 -it \
      --rm --restart=Never -- wget http://google.com --timeout=5 --tries=1
    wget: download timed out
    pod "web" deleted
    pod default/web terminated (Error)
    ```

    +   首先，请查看 Dashboard 是否已安装。您可以检查 Dashboard 通常创建的命名空间：

    ```
    $ kubectl get ns kubernetes-dashboard
    NAME                   STATUS   AGE
    kubernetes-dashboard   Active   109s
    ```

    如果命名空间不存在，则可以假定尚未安装 Dashboard。使用以下命令安装它：

    ```
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/\
    dashboard/v2.6.0/aio/deploy/recommended.yaml
    ```

    创建 ServiceAccount、ClusterRole 和 ClusterRoleBinding。确保 ClusterRole 只允许列出部署对象。以下 YAML 清单已保存在文件 `dashboard-observer-user.yaml` 中：

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: observer-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      name: cluster-observer
    rules:
    - apiGroups:
      - 'apps'
      resources:
      - 'deployments'
      verbs:
      - list
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: observer-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-observer
    subjects:
    - kind: ServiceAccount
      name: observer-user
      namespace: kubernetes-dashboard
    ```

    使用以下命令创建对象：

    ```
    $ kubectl apply -f dashboard-observer-user.yaml
    ```

    +   运行以下命令为 ServiceAccount 创建一个令牌。选项 `--duration 0s` 确保令牌永不过期。复制在命令的控制台输出中呈现的令牌：

    ```
    $ kubectl create token observer-user -n kubernetes-dashboard \
      --duration 0s
    eyJhbGciOiJSUzI1NiIsImtpZCI6Ik5lNFMxZ1...
    ```

    运行代理命令，并在浏览器中打开链接 [*http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy*](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy) ：

    ```
    $ kubectl proxy
    ```

    选择“Token”身份验证方法，并粘贴您之前复制的令牌。登录到 Dashboard。您应该看到只有部署对象是可列出的（见 图 A-1）。

    所有其他对象将显示“这里没有任何内容。” 图 A-2 显示 Pod 的列表。

    ![ckss aa01](img/ckss_aa01.png)

    ###### 图 A-1\. 允许部署的 Dashboard 视图

    ![ckss aa02](img/ckss_aa02.png)

    ###### 图 A-2\. 不允许查看 Pod 的 Dashboard 视图

    +   使用以下命令下载 API 服务器二进制文件：

    ```
    $ curl -LO "https://dl.k8s.io/v1.26.1/bin/linux/amd64/kube-apiserver"
    ```

    接下来，下载同一二进制文件的 SHA256 文件，但是不同版本。以下命令下载版本 1.23.1 的文件：

    ```
    $ curl -LO "https://dl.k8s.io/v1.23.1/bin/linux/amd64/\
    kube-apiserver.sha256"
    ```

    与校验和文件比较二进制文件的结果失败，因为版本不匹配：

    ```
    $ echo "$(cat kube-apiserver.sha256)  kube-apiserver" | shasum -a 256 \
      --check
    kube-apiserver: FAILED
    shasum: WARNING: 1 computed checksum did NOT match
    ```****

****# 第三章，“集群加固”

1.  使用 `openssl` 可执行文件创建私钥。提供一个富有表现力的文件名，例如 `jill.key`。 `-subj` 选项提供用户名（CN）和组（O）。以下命令使用用户名 `jill` 和名为 `observer` 的组：

    ```
    $ openssl genrsa -out jill.key 2048
    $ openssl req -new -key jill.key -out jill.csr -subj \
      "/CN=jill/O=observer"
    ```

    使用以下命令检索 CSR 文件内容的 base64 编码值。在下一步创建 CertificateSigningRequest 对象时将需要它：

    ```
    $ cat jill.csr | base64 | tr -d "\n"
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tL...
    ```

    以下脚本创建一个 CertificateSigningRequest 对象：

    ```
    $ cat <<EOF | kubectl apply -f -
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: jill
    spec:
      request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tL...
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400
      usages:
      - client auth
    EOF
    ```

    使用`certificate approve`命令批准签名请求并导出已颁发的证书：

    ```
    $ kubectl certificate approve jill
    $ kubectl get csr jill -o jsonpath=*{.status.certificate}*| base64 \
      -d > jill.crt
    ```

    将用户添加到 kubeconfig 文件，并为用户添加上下文。此处使用的集群名称是`minikube`。在您的 Kubernetes 环境中可能不同：

    ```
    $ kubectl config set-credentials jill --client-key=jill.key \
      --client-certificate=jill.crt --embed-certs=true
    $ kubectl config set-context jill --cluster=minikube --user=jill
    ```

1.  创建角色和角色绑定。以下命令将动词`get`、`list`和`watch`分配给`observer`组的`subject`，类型为组，用于 Pods、ConfigMaps 和 Secrets。用户`jill`是该组的一部分：

    ```
    $ kubectl create role observer --verb=create --verb=get --verb=list \
      --verb=watch --resource=pods --resource=configmaps --resource=secrets
    $ kubectl create rolebinding observer-binding --role=observer \
      --group=observer
    ```

1.  切换到用户上下文：

    ```
    $ kubectl config use-context jill
    ```

    我们将选择一个允许的操作，列出 ConfigMap 对象。用户被授权映射此调用：

    ```
    $ kubectl get configmaps
    NAME               DATA   AGE
    kube-root-ca.crt   1      16m
    ```

    列出节点将不被授权。用户没有适当的权限：

    ```
    $ kubectl get nodes
    Error from server (Forbidden): nodes is forbidden: User "jill" cannot \
    list resource "nodes" in API group "" at the cluster scope
    ```

    切换回管理员上下文：

    ```
    $ kubectl config use-context minikube
    ```

1.  创建命名空间`t23`：

    ```
    $ kubectl create namespace t23
    ```

    在命名空间中创建服务账户`api-call`：

    ```
    $ kubectl create serviceaccount api-call -n t23
    ```

    使用名为`pod.yaml`的 YAML 清单文件定义 Pod。文件内容定义一个 Pod，该 Pod 向 API 服务器发出 HTTPS GET 调用，以检索`default`命名空间中的 Service 列表：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: service-list
      namespace: t23
    spec:
      serviceAccountName: api-call
      containers:
      - name: service-list
        image: alpine/curl:3.14
        command: ['sh', '-c', 'while true; do curl -s -k -m 5 \
                  -H "Authorization: Bearer $(cat /var/run/secrets/\
                  kubernetes.io/serviceaccount/token)" https://kubernetes.\
                  default.svc.cluster.local/api/v1/namespaces/default/\
                  services; sleep 10; done']
    ```

    使用以下命令创建 Pod：

    ```
    $ kubectl apply -f pod.yaml
    ```

    检查 Pod 的日志。API 调用未经授权，如下所示的日志输出：

    ```
    $ kubectl logs service-list -n t23
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {},
      "status": "Failure",
      "message": "services is forbidden: User \"system:serviceaccount:t23 \
                  :api-call\" cannot list resource \"services\" in API \
                  group \"\" in the namespace \"default\"",
      "reason": "Forbidden",
      "details": {
        "kind": "services"
      },
      "code": 403
    }
    ```

1.  在名为`clusterrole.yaml`的文件中创建 YAML 清单，如下所示：

    ```
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: list-services-clusterrole
    rules:
    - apiGroups: [""]
      resources: ["services"]
      verbs: ["list"]
    ```

    在名为`rolebinding.yaml`的文件中引用 ClusterRole。主体应列出命名空间`t23`中的服务账户`api-call`：

    ```
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: serviceaccount-service-rolebinding
    subjects:
    - kind: ServiceAccount
      name: api-call
      namespace: t23
    roleRef:
      kind: ClusterRole
      name: list-services-clusterrole
      apiGroup: rbac.authorization.k8s.io
    ```

    从 YAML 清单创建这两个对象：

    ```
    $ kubectl apply -f clusterrole.yaml
    $ kubectl apply -f rolebinding.yaml
    ```

    容器内运行的 API 调用现在应已获得授权，并允许列出`default`命名空间中的 Service 对象。如下输出所示，该命名空间目前至少托管一个 Service 对象，即`kubernetes.default` Service：

    ```
    $ kubectl logs service-list -n t23
    {
      "kind": "ServiceList",
      "apiVersion": "v1",
      "metadata": {
        "resourceVersion": "1108"
      },
      "items": [
         {
           "metadata": {
             "name": "kubernetes",
             "namespace": "default",
             "uid": "30eb5425-8f60-4bb7-8331-f91fe0999e20",
             "resourceVersion": "199",
             "creationTimestamp": "2022-09-08T18:06:52Z",
             "labels": {
               "component": "apiserver",
               "provider": "kubernetes"
           },
           ...
         }
      ]
    }
    ```

1.  使用以下命令为服务账户创建令牌：

    ```
    $ kubectl create token api-call -n t23
    eyJhbGciOiJSUzI1NiIsImtpZCI6IjBtQkJzVWlsQjl...
    ```

    通过删除并重新创建实时对象更改现有 Pod 定义。添加禁用自动挂载令牌的属性，如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: service-list
      namespace: t23
    spec:
      serviceAccountName: api-call
      automountServiceAccountToken: false
      containers:
      - name: service-list
        image: alpine/curl:3.14
        command: ['sh', '-c', 'while true; do curl -s -k -m 5 \
                  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ij \
                  BtQkJzVWlsQjl" https://kubernetes.default.svc.cluster. \
                  local/api/v1/namespaces/default/services; sleep 10; \
                  done']
    ```

    API 服务器将允许使用服务账户令牌执行的 HTTPS 请求进行身份验证和授权：

    ```
    $ kubectl logs service-list -n t23
    {
      "kind": "ServiceList",
      "apiVersion": "v1",
      "metadata": {
        "resourceVersion": "81194"
      },
      "items": [
         {
           "metadata": {
             "name": "kubernetes",
             "namespace": "default",
             "uid": "30eb5425-8f60-4bb7-8331-f91fe0999e20",
             "resourceVersion": "199",
             "creationTimestamp": "2022-09-08T18:06:52Z",
             "labels": {
               "component": "apiserver",
               "provider": "kubernetes"
           },
           ...
         }
      ]
    }
    ```

1.  解决此示例练习需要很多手动步骤。以下命令不会呈现它们的输出。

    使用 Vagrant 打开控制平面节点的交互式 Shell：

    ```
    $ vagrant ssh kube-control-plane
    ```

    将`kubeadm`升级到版本 1.26.1 并应用：

    ```
    $ sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get \
      install -y kubeadm=1.26.1-00 && sudo apt-mark hold kubeadm
    $ sudo kubeadm upgrade apply v1.26.1
    ```

    排空节点，升级 kubelet 和`kubectl`，重新启动 kubelet，并解封节点：

    ```
    $ kubectl drain kube-control-plane --ignore-daemonsets
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubelet=1.26.1-00 kubectl=1.26.1-00
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart kubelet
    $ kubectl uncordon kube-control-plane
    ```

    节点的版本现在应为 v1.26.1。退出节点：

    ```
    $ kubectl get nodes
    $ exit
    ```

    使用 Vagrant 打开第一个工作节点的交互式 Shell。重复以下所有步骤以操作工作节点：

    ```
    $ vagrant ssh kube-worker-1
    ```

    将`kubeadm`升级到版本 1.26.1 并应用到节点：

    ```
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubeadm=1.26.1-00
    $ sudo kubeadm upgrade node
    ```

    通过下列步骤来排空节点、升级 kubelet 和`kubectl`、重启 kubelet 并解除节点的限制：

    ```
    $ kubectl drain kube-worker-1 --ignore-daemonsets
    $ sudo apt-get update && sudo apt-get install -y \
      --allow-change-held-packages kubelet=1.26.1-00 kubectl=1.26.1-00
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart kubelet
    $ kubectl uncordon kube-worker-1
    ```

    节点的版本现在应显示为 v1.26.1。退出节点：

    ```
    $ kubectl get nodes
    $ exit
    ```

# 第四章，“系统加固”

1.  使用以下命令进入工作节点的 Shell：

    ```
    $ vagrant ssh kube-worker-1
    ```

    确定暴露端口 21 的进程。一种方法是使用`lsof`命令。暴露端口的命令是`vsftpd`：

    ```
    $ sudo lsof -i :21
    COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
    vsftpd  10178 root    3u  IPv6  56850      0t0  TCP *:ftp (LISTEN)
    ```

    或者，您也可以使用`ss`命令，如下所示：

    ```
    $ sudo ss -at -pn '( dport = :21 or sport = :21 )'
    State   Recv-Q   Send-Q   Local Address:Port \
       Peer Address:Port   Process
    LISTEN  0   32   *:21 \
       *:*   users:(("vsftpd",pid=10178,fd=3))

    ```

    进程`vsftpd`已作为服务启动：

    ```
    $ sudo systemctl status vsftpd
    ● vsftpd.service - vsftpd FTP server
         Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; \
                 vendor preset: enabled)
         Active: active (running) since Thu 2022-10-06 14:39:12 UTC; \
                 11min ago
       Main PID: 10178 (vsftpd)
          Tasks: 1 (limit: 1131)
         Memory: 604.0K
         CGroup: /system.slice/vsftpd.service
                 └─10178 /usr/sbin/vsftpd /etc/vsftpd.conf

    Oct 06 14:39:12 kube-worker-1 systemd[1]: Starting vsftpd FTP server...
    Oct 06 14:39:12 kube-worker-1 systemd[1]: Started vsftpd FTP server.
    ```

    关闭服务并卸载包：

    ```
    $ sudo systemctl stop vsftpd
    $ sudo systemctl disable vsftpd
    $ sudo apt purge --auto-remove -y vsftpd
    ```

    检查端口时，您将看到它不再列出：

    ```
    $ sudo lsof -i :21
    ```

    退出节点：

    ```
    $ exit
    ```

1.  使用以下命令进入工作节点的 Shell：

    ```
    $ vagrant ssh kube-worker-1
    ```

    使用命令`sudo vim /etc/apparmor.d/network-deny`在`/etc/apparmor.d/network-deny`创建 AppArmor 配置文件。文件内容应如下所示：

    ```
    #include <tunables/global>

    profile network-deny flags=(attach_disconnected) {
      #include <abstractions/base>

      network,
    }
    ```

    运行以下命令强制执行 AppArmor 配置：

    ```
    $ sudo apparmor_parser /etc/apparmor.d/network-deny
    ```

    您不能修改现有的 Pod 对象以添加 AppArmor 的注释。您需要首先删除对象。将 Pod 的定义写入文件：

    ```
    $ kubectl get pod -o yaml > pod.yaml
    $ kubectl delete pod network-call
    ```

    编辑`pod.yaml`文件以添加 AppArmor 注释。对于相关的注释，使用容器名称`network-call`作为键后缀，并使用`localhost/network-deny`作为值。后缀`network-deny`指的是 AppArmor 配置文件的名称。稍作清理后，最终内容可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: network-call
      annotations:
        container.apparmor.security.beta.kubernetes.io/network-call: \
        localhost/network-deny
    spec:
      containers:
      - name: network-call
        image: alpine/curl:3.14
        command: ["sh", "-c", "while true; do ping -c 1 google.com; \
                  sleep 5; done"]
    ```

    从清单中创建 Pod。几秒钟后，Pod 应该转换为“运行”状态：

    ```
    $ kubectl create -f pod.yaml
    $ kubectl get pod network-call
    NAME           READY   STATUS    RESTARTS   AGE
    network-call   1/1     Running   0          27s
    ```

    AppArmor 阻止 Pod 进行网络调用。您可以检查日志以进行验证：

    ```
    $ kubectl logs network-call
    ...
    sh: ping: Permission denied
    sh: sleep: Permission denied
    ```

    退出节点：

    ```
    $ exit
    ```

1.  使用以下命令进入工作节点的 Shell：

    ```
    $ vagrant ssh kube-worker-1
    ```

    为 seccomp 配置创建目标目录：

    ```
    $ sudo mkdir -p /var/lib/kubelet/seccomp/profiles
    ```

    在目录`/var/lib/kubelet/seccomp/profiles`中添加文件`audit.json`，内容如下：

    ```
    {
        "defaultAction": "SCMP_ACT_LOG"
    }
    ```

    您不能修改现有的 Pod 对象以通过安全上下文添加 seccomp 配置。您需要首先删除对象。将 Pod 的定义写入文件：

    ```
    $ kubectl get pod -o yaml > pod.yaml
    $ kubectl delete pod network-call
    ```

    编辑`pod.yaml`文件。将 seccomp 配置指向定义。稍作清理后，最终内容可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: network-call
    spec:
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/audit.json
      containers:
      - name: network-call
        image: alpine/curl:3.14
        command: ["sh", "-c", "while true; do ping -c 1 google.com; \
                  sleep 5; done"]
        securityContext:
          allowPrivilegeEscalation: false
    ```

    从清单中创建 Pod。几秒钟后，Pod 应该转换为“运行”状态：

    ```
    $ kubectl create -f pod.yaml
    $ kubectl get pod network-call
    NAME           READY   STATUS    RESTARTS   AGE
    network-call   1/1     Running   0          27s
    ```

    你应该能够找到系统调用的日志条目，例如`sleep`命令：

    ```
    $ sudo cat /var/log/syslog
    Oct  6 16:25:06 ubuntu-focal kernel: [ 2114.894122] audit: type=1326 \
    audit(1665073506.099:23761): auid=4294967295 uid=0 gid=0 \
    ses=4294967295 pid=19226 comm="sleep" exe="/bin/busybox" \
    sig=0 arch=c000003e syscall=231 compat=0 ip=0x7fc026adbf0b \
    code=0x7ffc0000
    ```

    退出节点：

    ```
    $ exit
    ```

    在文件`pod.yaml`中定义 Pod：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: sysctl-pod
    spec:
      securityContext:
        sysctls:
        - name: net.core.somaxconn
          value: "1024"
        - name: debug.iotrace
          value: "1"
      containers:
      - name: nginx
        image: nginx:1.23.1
    ```

    创建 Pod，然后检查状态。您将看到状态为“SysctlForbidden”：

    ```
    $ kubectl create -f pod.yaml
    $ kubectl get pods
    NAME         READY   STATUS            RESTARTS   AGE
    sysctl-pod   0/1     SysctlForbidden   0          4s
    ```

    事件日志将告诉您更多的原因：

    ```
    $ kubectl describe pod sysctl-pod
    ...
    Events:
      Type     Reason           Age    From    \
                 Message
      ----     ------           ----   ----    \
                 -------
      Warning  SysctlForbidden  2m48s  kubelet \
                 forbidden sysctl: "net.core.somaxconn" \
                 not allowlisted
    ```

# 第五章，“最小化微服务的漏洞”

1.  使用文件`busybox-security-context.yaml`中的安全设置定义 Pod。您可以找到以下 YAML 清单的内容：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox-security-context
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      volumes:
      - name: vol
        emptyDir: {}
      containers:
      - name: busybox
        image: busybox:1.28
        command: ["sh", "-c", "sleep 1h"]
        volumeMounts:
        - name: vol
          mountPath: /data/test
        securityContext:
          allowPrivilegeEscalation: false
    ```

    使用以下命令创建 Pod：

    ```
    $ kubectl apply -f busybox-security-context.yaml
    $ kubectl get pod busybox-security-context
    NAME                       READY   STATUS    RESTARTS   AGE
    busybox-security-context   1/1     Running   0          54s
    ```

    进入容器并创建文件。您会发现，文件组由安全上下文定义为 2000：

    ```
    $ kubectl exec busybox-security-context -it -- /bin/sh
    / $ cd /data/test
    /data/test $ touch hello.txt
    /data/test $ ls -l
    total 0
    -rw-r--r--    1 1000     2000             0 Nov 21 18:29 hello.txt
    /data/test $ exit
    ```

1.  在文件 `psa-namespace.yaml` 中指定名为 `audited` 的命名空间。使用 `baseline` 级别设置 PSA 标签和 `warn` 模式：

    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      name: audited
      labels:
        pod-security.kubernetes.io/warn: baseline
    ```

    从 YAML 清单创建命名空间：

    ```
    $ kubectl apply -f psa-namespace.yaml
    ```

    您可以在文件 `psa-pod.yaml` 中使用以下 Pod 配置产生错误。YAML 清单设置了属性 `hostNetwork: true`，这不允许 [`baseline` 级别](https://oreil.ly/c8JEW) 使用：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: audited
    spec:
      hostNetwork: true
      containers:
      - name: busybox
        image: busybox:1.28
        command: ["sh", "-c", "sleep 1h"]
    ```

    创建 Pod 会显示警告消息。尽管如此，Pod 已创建。通过配置 PSA 使用 `restricted` 级别可以防止 Pod 的创建：

    ```
    $ kubectl apply -f psa-pod.yaml
    Warning: would violate PodSecurity "baseline:latest": host namespaces \
    (hostNetwork=true)
    pod/busybox created
    $ kubectl get pod busybox -n audited
    NAME      READY   STATUS    RESTARTS   AGE
    busybox   1/1     Running   0          2m21s
    ```

1.  您可以使用以下命令安装 Gatekeeper：

    ```
    $ kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/\
    gatekeeper/master/deploy/gatekeeper.yaml
    ```

    Gatekeeper 库描述了用于定义 [副本限制](https://oreil.ly/gyD1-) 的 ConstraintTemplate。检查页面描述的 YAML 清单。使用以下命令应用清单：

    ```
    $ kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/\
    gatekeeper-library/master/library/general/replicalimits/template.yaml
    ```

    现在，在名为 `replica-limits-constraint.yaml` 的文件中定义 Constraint 的 YAML 清单：

    ```
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sReplicaLimits
    metadata:
      name: replica-limits
    spec:
      match:
        kinds:
          - apiGroups: ["apps"]
            kinds: ["Deployment"]
      parameters:
        ranges:
        - min_replicas: 3
          max_replicas: 10
    ```

    使用以下命令创建 Constraint：

    ```
    $ kubectl apply -f replica-limits-constraint.yaml
    ```

    您可以看到，只有在提供的副本数目在 Constraint 的范围内时，才能创建 Deployment：

    ```
    $ kubectl create deployment nginx --image=nginx:1.23.2 --replicas=15
    error: failed to create deployment: admission webhook \
    "validation.gatekeeper.sh" denied the request: [replica-limits] \
    The provided number of replicas is not allowed for deployment: nginx. \
    Allowed ranges: {"ranges": [{"max_replicas": 10, "min_replicas": 3}]}
    $ kubectl create deployment nginx --image=nginx:1.23.2 --replicas=7
    deployment.apps/nginx created
    ```

1.  配置 etcd 的加密，如 “加密 etcd 数据” 中所述。接下来，使用以下命令创建新的 Secret：

    ```
    $ kubectl create secret generic db-credentials \
      --from-literal=api-key=YZvkiWUkycvspyGHk3fQRAkt
    ```

    您可以使用以下命令检查存储在 etcd 中的 Secret 的加密值：

    ```
    $ sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/\
    etcd/server.key get /registry/secrets/default/db-credentials | hexdump -C
    ```

1.  使用 Vagrant 打开工作节点的交互式 shell：

    ```
    $ vagrant ssh kube-worker-1
    ```

    使用以下 YAML 清单定义 RuntimeClass。内容已存储在文件 `runtime-class.yaml` 中：

    ```
    apiVersion: node.k8s.io/v1
    kind: RuntimeClass
    metadata:
      name: container-runtime-sandbox
    handler: runsc
    ```

    创建 RuntimeClass 对象：

    ```
    $ kubectl apply -f runtime-class.yaml
    ```

    使用 `spec.runtimeClassName` 属性将 RuntimeClass 的名称分配给 Pod。nginx Pod 已在文件 `pod.yaml` 中定义：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
    spec:
      runtimeClassName: container-runtime-sandbox
      containers:
      - name: nginx
        image: nginx:1.23.2
    ```

    创建 Pod 对象。Pod 将转换为状态“Running”：

    ```
    $ kubectl apply -f pod.yaml
    $ kubectl get pod nginx
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          2m21s
    ```

    退出节点：

    ```
    $ exit
    ```

# 第六章，“供应链安全”

1.  使用提供的 Dockerfile 构建的初始容器映像大小为 998MB。您可以使用以下命令生成并运行容器映像。运行快速 `curl` 命令查看应用程序暴露的端点是否可达：

    ```
    $ docker build . -t node-app:0.0.1
    ...
    $ docker images
    REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
    node-app     0.0.1     7ba99d4ba3af   3 seconds ago   998MB
    $ docker run -p 3001:3001 -d node-app:0.0.1
    c0c8a301eeb4ac499c22d10399c424e1063944f18fff70ceb5c49c4723af7969
    $ curl -L http://localhost:3001/
    Hello World
    ```

    您可以进行的更改之一是避免使用大型基础映像。您可以将其替换为 `alpine` 版本的 node 基础映像。还要避免拉取 `latest` 映像。选择您实际希望应用程序运行的 Node.js 版本。以下命令使用带有基础映像 `node:19-alpine` 的 Dockerfile，将容器映像大小减小到 176MB：

    ```
    $ docker build . -t node-app:0.0.1
    ...
    $ docker images
    REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
    node-app     0.0.1     ef2fbec41a75   2 seconds ago    176MB
    ```

1.  您可以使用 Helm 安装 Kyverno，也可以指向项目 GitHub 仓库中可用的 YAML 清单。我们将在这里使用 YAML 清单：

    ```
    $ kubectl create -f https://raw.githubusercontent.com/kyverno/\
    kyverno/main/config/install.yaml
    ```

    设置名为`restrict-image-registries.yaml`的 YAML 清单文件。向文件添加以下内容。该清单代表一个 ClusterPolicy，仅允许使用以`gcr.io/`开头的容器镜像。确保将`spec.validationFailureAction`属性的值设为`Enforce`：

    ```
    apiVersion: kyverno.io/v1
    kind: ClusterPolicy
    metadata:
      name: restrict-image-registries
      annotations:
        policies.kyverno.io/title: Restrict Image Registries
        policies.kyverno.io/category: Best Practices, EKS Best Practices
        policies.kyverno.io/severity: medium
        policies.kyverno.io/minversion: 1.6.0
        policies.kyverno.io/subject: Pod
        policies.kyverno.io/description: >-
          Images from unknown, public registries can be of dubious quality \
          and may not be scanned and secured, representing a high degree of \
          risk. Requiring use of known, approved registries helps reduce \
          threat exposure by ensuring image pulls only come from them. This \
          policy validates that container images only originate from the \
          registry `eu.foo.io` or `bar.io`. Use of this policy requires \
          customization to define your allowable registries.
    spec:
      validationFailureAction: Enforce
      background: true
      rules:
      - name: validate-registries
        match:
          any:
          - resources:
              kinds:
              - Pod
        validate:
          message: "Unknown image registry."
          pattern:
            spec:
              containers:
              - image: "gcr.io/*"
    ```

    使用以下命令应用清单：

    ```
    $ kubectl apply -f restrict-image-registries.yaml
    ```

    运行以下命令验证策略是否已生效。任何不使用前缀`gcr.io/`的容器镜像定义将被拒绝：

    ```
    $ kubectl run nginx --image=nginx:1.23.3
    Error from server: admission webhook "validate.kyverno.svc-fail" \
    denied the request:

    policy Pod/default/nginx for resource violation:

    restrict-image-registries:
      validate-registries: 'validation error: Unknown image registry. \
      rule validate-registries
        failed at path /spec/containers/0/image/'
    $ kubectl run busybox --image=gcr.io/google-containers/busybox:1.27.2
    pod/busybox created
    ```

1.  使用 Docker Hub 的搜索功能找到镜像`nginx:1.23.3-alpine`的 SHA256 哈希。[搜索结果](https://oreil.ly/a4o8E)将引导您到镜像的标签。在页面顶部，您应该找到摘要`sha256:c1b9fe3c0c015486cf1e4a0ecabe78d05864475e279638e9713eb55f013f907f`。在 Pod 定义中使用摘要而不是标签。结果如下所示的 YAML 清单：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx@sha256:c1b9fe3c0c015486cf1e4a0ecabe78d05864475e279638 \
               e9713eb55f013f907f
    ```

    应该正常创建 Pod：

    ```
    $ kubectl apply -f pod-validate-image.yaml
    pod/nginx created
    $ kubectl get pods nginx
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          29s
    ```

    如果以任何形式修改 SHA256 哈希并尝试重新创建 Pod，则 Kubernetes 将不允许您拉取镜像。

1.  在 Docker 容器中运行 Kubesec 将产生一堆建议，如下所示的输出：

    ```
    $ docker run -i kubesec/kubesec:512c5e0 scan /dev/stdin < pod.yaml
    [
      {
        "object": "Pod/hello-world.default",
        "valid": true,
        "message": "Passed with a score of 0 points",
        "score": 0,
        "scoring": {
          "advise": [
            {
              "selector": "containers[] .securityContext .capabilities \
                           .drop | index(\"ALL\")",
              "reason": "Drop all capabilities and add only those \
                         required to reduce syscall attack surface"
            },
            {
              "selector": "containers[] .resources .requests .cpu",
              "reason": "Enforcing CPU requests aids a fair balancing \
                         of resources across the cluster"
            },
            {
              "selector": "containers[] .securityContext .runAsNonRoot \
                           == true",
              "reason": "Force the running image to run as a non-root \
                         user to ensure least privilege"
            },
            {
              "selector": "containers[] .resources .limits .cpu",
              "reason": "Enforcing CPU limits prevents DOS via resource \
                         exhaustion"
            },
            {
              "selector": "containers[] .securityContext .capabilities \
                           .drop",
              "reason": "Reducing kernel capabilities available to a \
                         container limits its attack surface"
            },
            {
              "selector": "containers[] .resources .requests .memory",
              "reason": "Enforcing memory requests aids a fair balancing \
                         of resources across the cluster"
            },
            {
              "selector": "containers[] .resources .limits .memory",
              "reason": "Enforcing memory limits prevents DOS via resource \
                         exhaustion"
            },
            {
              "selector": "containers[] .securityContext \
                           .readOnlyRootFilesystem == true",
              "reason": "An immutable root filesystem can prevent malicious \
                         binaries being added to PATH and increase attack \
                         cost"
            },
            {
              "selector": ".metadata .annotations .\"container.seccomp. \
                           security.alpha.kubernetes.io/pod\"",
              "reason": "Seccomp profiles set minimum privilege and secure \
                         against unknown threats"
            },
            {
              "selector": ".metadata .annotations .\"container.apparmor. \
                           security.beta.kubernetes.io/nginx\"",
              "reason": "Well defined AppArmor policies may provide greater \
                         protection from unknown threats. WARNING: NOT \
                         PRODUCTION READY"
            },
            {
              "selector": "containers[] .securityContext .runAsUser -gt \
                           10000",
              "reason": "Run as a high-UID user to avoid conflicts with \
                         the host's user table"
            },
            {
              "selector": ".spec .serviceAccountName",
              "reason": "Service accounts restrict Kubernetes API access \
                         and should be configured with least privilege"
            }
          ]
        }
      }
    ]
    ```

    修正后的 YAML 清单可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: hello-world
    spec:
      serviceAccountName: default
      containers:
      - name: linux
        image: hello-world:linux
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 20000
          capabilities:
            drop: ["ALL"]
    ```

1.  对现有的`setup.yaml`清单执行`kubectl apply`命令将在命名空间`r61`中创建名为`backend`、`loop`和`logstash`的 Pod：

    ```
    $ kubectl apply -f setup.yaml
    namespace/r61 created
    pod/backend created
    pod/loop created
    pod/logstash created
    ```

    您可以使用以下命令查看它们：

    ```
    $ kubectl get pods -n r61
    NAME       READY   STATUS    RESTARTS   AGE
    backend    1/1     Running   0          115s
    logstash   1/1     Running   0          115s
    loop       1/1     Running   0          115s
    ```

    使用`kubectl describe`命令检查命名空间`r61`中每个 Pod 的镜像。使用的镜像是`bmuschko/nodejs-hello-world:1.0.0`，`alpine:3.13.4`和`elastic/logstash:7.13.3`：

    ```
    $ kubectl describe pod backend -n r61
    ...
    Containers:
      hello:
        Container ID:   docker://eb0bdefc75e635d03b625140d1e \
                        b229ca2db7904e44787882147921c2bd9c365
        Image:          bmuschko/nodejs-hello-world:1.0.0
        ...
    ```

    使用 Trivy 可执行文件检查所有镜像的漏洞：

    ```
    $ trivy image bmuschko/nodejs-hello-world:1.0.0
    $ trivy image alpine:3.13.4
    $ trivy image elastic/logstash:7.13.3
    ```

    如果仔细查看漏洞列表，您会发现所有镜像都包含“CRITICAL”严重性的问题。因此，删除所有 Pod：

    ```
    $ kubectl delete pod backend -n r61
    $ kubectl delete pod logstash -n r61
    $ kubectl delete pod loop -n r61
    ```

# 第七章，“监控、日志和运行时安全”

1.  使用以下命令进入工作节点的 Shell：

    ```
    $ vagrant ssh kube-worker-1
    ```

    检查名为`malicious`的运行中 Pod 的命令和参数。您将看到它试图将消息追加到文件`/etc/threat`：

    ```
    $ kubectl get pod malicious -o jsonpath='{.spec.containers[0].args}'
    ...
    spec:
      containers:
      - args:
        - /bin/sh
        - -c
        - while true; do echo "attacker intrusion" >> /etc/threat; \
          sleep 5; done
    ...

    ```

    Falco 的默认规则之一监视尝试向`/etc`目录写入的文件操作。每次写入尝试都会在标准输出中找到一条消息：

    ```
    $ sudo journalctl -fu falco
    Jan 24 23:40:18 kube-worker-1 falco[8575]: 23:40:18.359740123: Error \
    File below /etc opened for writing (user=<NA> user_loginuid=-1 \
    command=sh -c while true; do echo "attacker intrusion" >> /etc/threat; \
    sleep 5; done pid=9763 parent=<NA> pcmdline=<NA> file=/etc/threat \
    program=sh gparent=<NA> ggparent=<NA> gggparent=<NA> \
    container_id=e72a6dbb63b8 image=docker.io/library/alpine)
    ...
    ```

    通过搜索字符串“etc opened for writing.”在`/etc/falco/falco_rules.yaml`中找到生成消息的规则。规则如下所示：

    ```
    - rule: Write below etc
      desc: an attempt to write to any file below /etc
      condition: write_etc_common
      output: "File below /etc opened for writing (user=%user.name \
               user_loginuid=%user.loginuid command=%proc.cmdline \
               pid=%proc.pid parent=%proc.pname pcmdline=%proc.pcmdline \
               file=%fd.name program=%proc.name gparent=%proc.aname[2] \
               ggparent=%proc.aname[3] gggparent=%proc.aname[4] \
               container_id=%container.id image=%container.image.repository)"
      priority: ERROR
      tags: [filesystem, mitre_persistence]
    ```

    将规则复制到文件`/etc/falco/falco_rules.local.yaml`并修改输出定义，如下所示：

    ```
    - rule: Write below etc
      desc: an attempt to write to any file below /etc
      condition: write_etc_common
      output: "%evt.time,%user.name,%container.id"
      priority: ERROR
      tags: [filesystem, mitre_persistence]
    ```

    重新启动 Falco 服务，并在 Falco 日志中找到更改的输出：

    ```
    $ sudo systemctl restart falco
    $ sudo journalctl -fu falco
    Jan 24 23:48:18 kube-worker-1 falco[17488]: 23:48:18.516903001: \
    Error 23:48:18.516903001,<NA>,e72a6dbb63b8
    ...
    ```

    编辑文件`/etc/falco/falco.yaml`以更改输出通道。禁用标准输出，启用文件输出，并将`file_output`属性指向文件`/var/log/falco.log`。结果配置如下所示：

    ```
    file_output:
      enabled: true
      keep_alive: false
      filename: /var/log/falco.log

    stdout_output:
      enabled: false
    ```

    现在日志文件将附加 Falco 日志：

    ```
    $ sudo tail -f /var/log/falco.log
    00:10:30.425084165: Error 00:10:30.425084165,<NA>,e72a6dbb63b8
    ...
    ```

    退出虚拟机：

    ```
    $ exit
    ```

1.  从`setup.yaml`文件创建名为`hash`的 Pod。其容器中运行的命令会在`/var/config/hash.txt`路径的文件末尾添加一个哈希值，以无限循环方式执行：

    ```
    $ kubectl apply -f setup.yaml
    pod/hash created
    $ kubectl get pod hash
    NAME   READY   STATUS    RESTARTS   AGE
    hash   1/1     Running   0          27s
    $ kubectl exec -it hash -- /bin/sh
    / # ls /var/config/hash.txt
    /var/config/hash.txt
    ```

    要使容器不可变，您需要在现有 Pod 定义中添加配置。您必须将根文件系统设置为只读访问，并将一个卷挂载到路径`/var/config`以允许向名为`hash.txt`的文件写入。生成的 YAML 清单可能如下所示：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: hash
    spec:
      containers:
      - name: hash
        image: alpine:3.17.1
        securityContext:
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: hash-vol
          mountPath: /var/config
        command: ["sh", "-c", "if [ ! -d /var/config ]; then mkdir -p \
                  /var/config; fi; while true; do echo $RANDOM | md5sum \
                  | head -c 20 >> /var/config/hash.txt; sleep 20; done"]
      volumes:
      - name: hash-vol
        emptyDir: {}
    ```

1.  使用以下命令进入控制平面节点的 Shell：

    ```
    $ vagrant ssh kube-control-plane
    ```

    编辑现有的审计策略文件`/etc/kubernetes/audit/rules/audit-policy.yaml`。根据说明添加所需的规则。最终审计策略文件的内容可能如下所示：

    ```
    apiVersion: audit.k8s.io/v1
    kind: Policy
    omitStages:
      - "RequestReceived"
    rules:
      - level: RequestResponse
        resources:
        - group: ""
          resources: ["pods"]
      - level: Metadata
        resources:
        - group: ""
          resources: ["secrets", "configmaps"]
      - level: Request
        resources:
        - group: ""
          resources: ["services"]
    ```

    通过编辑文件`/etc/kubernetes/manifests/kube-apiserver.yaml`，配置 API 服务器以使用审计策略文件。根据请求提供额外选项。所需的相关配置如下：

    ```
    ...
    spec:
      containers:
      - command:
        - kube-apiserver
        - --audit-policy-file=/etc/kubernetes/audit/rules/audit-policy.yaml
        - --audit-log-path=/var/log/kubernetes/audit/logs/apiserver.log
        - --audit-log-maxage=5
        ...
        volumeMounts:
        - mountPath: /etc/kubernetes/audit/rules/audit-policy.yaml
          name: audit
          readOnly: true
        - mountPath: /var/log/kubernetes/audit/logs/
          name: audit-log
          readOnly: false
      ...
      volumes:
      - name: audit
        hostPath:
          path: /etc/kubernetes/audit/rules/audit-policy.yaml
          type: File
      - name: audit-log
        hostPath:
          path: /var/log/kubernetes/audit/logs/
          type: DirectoryOrCreate
    ```

    日志记录的资源之一是`Metadata`级别的 ConfigMap。以下命令创建一个示例 ConfigMap 对象：

    ```
    $ kubectl create configmap db-user --from-literal=username=tom
    configmap/db-user created
    ```

    现在，审计日志文件将包含此事件的条目：

    ```
    $ sudo cat /var/log/kubernetes/audit/logs/apiserver.log
    {"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata", \
    "auditID":"1fbb409a-3815-4da8-8a5e-d71c728b98b1","stage": \
    "ResponseComplete","requestURI":"/api/v1/namespaces/default/configmaps? \
    fieldManager=kubectl-create\u0026fieldValidation=Strict","verb": \
    "create","user":{"username":"kubernetes-admin","groups": \
    ["system:masters","system:authenticated"]},"sourceIPs": \
    ["192.168.56.10"], "userAgent":"kubectl/v1.24.4 (linux/amd64) \
    kubernetes/95ee5ab", "objectRef":{"resource":"configmaps", \
    "namespace":"default", "name":"db-user","apiVersion":"v1"}, \
    "responseStatus":{"metadata": {},"code":201}, \
    "requestReceivedTimestamp":"2023-01-25T18:57:51.367219Z", \
    "stageTimestamp":"2023-01-25T18:57:51.372094Z","annotations": \
    {"authorization.k8s.io/decision":"allow", \
    "authorization.k8s.io/reason":""}}
    ```

    退出虚拟机：

    ```
    $ exit
    ```****
