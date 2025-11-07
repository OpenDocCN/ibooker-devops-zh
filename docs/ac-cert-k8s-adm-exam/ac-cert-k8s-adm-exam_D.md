# 附录 D. 解决考试练习题

本附录将按章节顺序向您介绍解决考试练习题的方法，从第一章开始。这将帮助您进入正确的思维模式，以启动 CKA 考试任务的可能解决方案。如果您已经阅读了本书，解决方案应该是显而易见的，但本附录不会立即向您提供答案，目的是为了帮助您为考试做准备。

## D.1 第一章考试练习

本章的考试练习题比其他章节少，因为这是一章介绍性章节。练习题更具探索性，我们将在这里进行回顾。

### D.1.1 列出 API 资源

当在集群内部列出 API 资源时，您应该首先考虑使用`kubectl`命令行工具。这无疑是 easiest 方法。如果您手头不知道命令，您总是可以通过简单地输入`kubectl`来使用帮助菜单，这将提供一些线索。帮助菜单的输出应该类似于以下内容（已缩写）：

```
root@kind-control-plane:/# kubectl
kubectl controls the Kubernetes cluster manager.

 Find more information at: https://kubernetes.io/docs/reference/kubectl/

Basic Commands (Beginner):
  create          Create a resource from a file or from stdin
  run             Run a particular image on the cluster
  set             Set specific features on objects

...

Other Commands:
  alpha           Commands for features in alpha
  api-resources   Print the supported API resources on the server
  api-versions    Print the supported API versions on the server, in the 
➥ form of "group/version"
  config          Modify kubeconfig files
  plugin          Provides utilities for interacting with plugins
  version         Print the client and server version information

Usage:
  kubectl [flags] [options]
```

如果您查看帮助菜单中名为“其他命令”的部分，您会看到“打印服务器上支持的 API 资源”的命令是`api-resources`。因此，解决这个练习的命令是`kubectl api-resources`。

### D.1.2 列出服务

在第 1.7 节中介绍了如何在 Linux 操作系统上列出集群中的服务，这与我们在第七章中讨论的 Kubernetes 服务不同。每当您看到术语*Linux*、*守护进程*或*系统服务*时，您应该考虑节点本身上的系统组件，而在 Kubernetes 的上下文中，服务是完全不同的资源。要列出与 Kubernetes 相关的节点上的服务，我们运行命令`systemctl list-unit-files --type service --all`，然后我们可以使用 Linux 中的 grep 功能进一步搜索`list-unit-files`命令的结果。完整的命令将类似于以下内容：

```
root@kind-control-plane:/# systemctl list-unit-files --type service --all | 
➥ grep kube
kubelet.service                        enabled         enabled
```

系统服务很多，这就是我们为什么使用 grep 功能只列出我们需要的那个，即 kubelet。在 CKA 考试的上下文中，kubelet 是唯一位于节点本身上的系统服务。要将输出发送到名为`services.csv`的文件中，我们可以在现有命令中添加`> services.csv`。因此，完整的命令将是`systemctl list-unit-files --type service --all | grep kubelet > services.csv`。

### D.1.3 kubelet 服务的状态

对于这个考试练习，适用的规则与上一个练习相同。kubelet 服务将是节点本身运行的唯一系统服务。当你考虑对 kubelet 服务进行任何操作时，请考虑使用 `systemctl`。你已经在之前的练习中看到了它的使用。`systemctl` 是控制 systemd 的标准实用工具，systemd 是控制所有系统服务的总服务。获取任何 systemd 服务状态的命令是 `systemctl status`。你可以在帮助菜单中找到这个提示，所以如果你在考试中完全忘记了，也不要担心。（让帮助菜单成为你的朋友，使用命令 `systemd -h`。）因此，完整的命令是 `systemctl status kubelet`。使用上一次练习中的相同方法，我们将输出到名为 `kubelet-status.txt` 的文件中，完整的命令是 `systemctl status kubelet > kubelet-status.txt`。

### D.1.4 使用声明式语法

正如我们在第 1.8 节中学到的，声明式语法有助于保留我们的 Kubernetes 配置的历史记录。与命令式不同，命令式是一系列按顺序执行的命令，声明式允许我们定义配置的最终状态，Kubernetes 控制器将使其成为现实。要创建一个包含 Pod 规范的 YAML 文件，我们可以使用 Vim 文本编辑器。

使用命令 `vim chap1-pod.yaml` 创建并打开文件。一旦文件在 Vim 中打开，你就可以编写 YAML。作为替代方案，并且这是我多次在书中推荐你做的，以节省你在考试中的时间，你可以让 `kubectl` 命令行为你编写 YAML，使用命令 `k run pod --image nginx --dry-run=client -o yaml > chap1-pod.yaml`。结果将是以下内容：

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod
  name: pod
spec:
  containers:
  - image: nginx
    name: pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

无论你选择哪种方式来完成这个练习都行，但要知道后者是一个我推荐在考试中使用的快捷方式。

### D.1.5 列出 Kubernetes 服务

这是我们区分系统服务（作为节点上 Linux 的一部分）和我们所说的 Kubernetes 服务的地方。正如练习中提到的，列出你在 Kubernetes 集群中创建的服务，其中“在 Kubernetes 集群中创建”是引导你使用 `kubectl` 工具而不是 `systemctl` 工具的关键词。

要列出服务，我们可以使用 `kubectl` 帮助菜单来找到正确的命令。就像许多其他会列出 Kubernetes 资源的操作一样，将其视为对 API 的 `GET` 请求。我们正在列出 API 中的内容，并且使用 `kubectl` 工具来做这件事。我们列出集群中所有内容的方法是附加 `--all-namespaces` 或简写为 `-A`。因此，完整的命令是 `kubectl get svc -A`，或者它也可能是另一个答案，即 `kubectl get services --all-namespaces`。输出将类似于以下内容：

```
root@kind-control-plane:~# kubectl get svc -A
NAMESPACE     NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   
➥ PORT(S)                  AGE
default       kubernetes       ClusterIP   10.96.0.1      <none>        
➥ 443/TCP                  56d
kube-system   kube-dns         ClusterIP   10.96.0.10     <none>        
➥ 53/UDP,53/TCP,9153/TCP   56d
```

## D.2 第二章考试练习

这些练习题位于第二章的末尾。练习题的数量与上一章大致相同，因此我们将以相同的目的进行练习。

### D.2.1 缩短 kubectl 命令

我们在第一章中简要地讨论了这个问题，但在考试中别名已经为您设置好了。这是一个您需要为将要使用的本地集群（以及您在工作中将要使用的集群）进行练习的练习。设置别名相当直接；然而，有两种方法可以做到这一点，因为执行命令`alias k=kubectl`是正确的；这将在您退出当前的 Bash 会话时重置。为了使其持久，我们可以使用命令`echo "alias k=kubectl" >> ~/.bashrc`将其添加到您的 Bash 配置文件中。这是一个更持久的命令，因为您可以注销并重新登录到您的机器，别名将保持不变。

### D.2.2 列出正在运行的 Pod

我们已经到达了将要使用`kubectl`命令行工具的阶段，您应该非常熟悉这个工具。您将在尝试解决考试中的几乎所有任务时使用它。要列出`kube-system`命名空间中运行的 Pod，您应该想到关键字`get`，因为它是`kubectl`（或别名`k`）在列出大多数 Kubernetes 资源后常见的单词。此外，这个练习要求我们显示 Pod IP 地址，因此我们正在寻找详细输出。在显示 Pod 和节点的 IP 地址时，考虑“输出宽”。当您将这些全部组合起来，完整的命令是`k get po -n kube-system -o wide`。输出应该看起来类似于以下内容：

```
root@kind-control-plane:~# k get po -n kube-system -o wide
NAME                                         READY   STATUS    RESTARTS       
➥ AGE   IP            NODE
coredns-565d847f94-75lsz                     1/1     Running   7 (28h ago)    
➥ 56d   10.244.0.11   kind-control-plane
coredns-565d847f94-8stkp                     1/1     Running   7 (28h ago)    
➥ 56d   10.244.0.7    kind-control-plane
etcd-kind-control-plane                      1/1     Running   8 (28h ago)    
➥ 56d   172.18.0.2    kind-control-plane
kindnet-b9f9r                                1/1     Running   27 (28h ago)   
➥ 56d   172.18.0.2    kind-control-plane
kube-apiserver-kind-control-plane            1/1     Running   5 (28h ago)    
➥ 52d   172.18.0.2    kind-control-plane
kube-controller-manager-kind-control-plane   1/1     Running   19 (67m ago)   
➥ 56d   172.18.0.2    kind-control-plane
kube-proxy-ddnwz                             1/1     Running   4 (28h ago)    
➥ 49d   172.18.0.2    kind-control-plane
kube-scheduler-kind-control-plane            1/1     Running   14 (67m ago)   
➥ 52d   172.18.0.2    kind-control-plane
metrics-server-d5589cfb-4ksgb                1/1     Running   8 (28h ago)    
➥ 52d   10.244.0.10   kind-control-plane
```

现在您已经得到了输出结果，您可以将它保存到名为`pod-ip-output.txt`的文件中，使用命令`k get po -n kube-system -o wide > pod-ip-output.txt`。

### D.2.3 查看 kubelet 客户端证书

我们在本章中了解到，Kubernetes 中的每个组件都有一个证书授权机构以及客户端证书或服务器证书。这就是客户端-服务器模型的工作方式，最常见的一个例子就是万维网。

当您想到证书时，您应该总是想到`/etc/kubernetes`目录，因为它存储了所有的证书文件和配置。无论是位于`/etc/kubernetes/pki`还是`/etc/kubernetes/pki/etcd`，您都会找到适当的证书文件，因为它们都有适当的标签。在这种情况下，从控制平面节点，我们可以使用命令`cd /etc/Kubernetes/pki`来更改目录。如果我们使用`ls`命令列出内容，我们会得到以下类似的输出：

```
root@kind-control-plane:/# cd /etc/kubernetes/pki
root@kind-control-plane:/etc/kubernetes/pki# ls
apiserver-etcd-client.crt  apiserver-kubelet-client.crt  apiserver.crt  
➥ ca.crt  etcd                front-proxy-ca.key      front-proxy-
➥ client.key  sa.pub
apiserver-etcd-client.key  apiserver-kubelet-client.key  apiserver.key  
➥ ca.key  front-proxy-ca.crt  front-proxy-client.crt  sa.key
```

结果，正如您能解读的，是 kubelet 客户端证书的位置，您可以使用命令`echo “/etc/kubernetes/pki” > kubelet-config.txt`将其输出到名为`kubelet-config.txt`的文件中。

### D.2.4 备份 etcd

如我们从第二章的阅读中了解到的，etcd 是集群所有配置的数据存储。它是跟踪在 `kube-system` 命名空间中运行的 Pods 的工具，同时也保留关于 kubelet 配置的数据。它维护自己的服务器证书，因为 API 服务器必须对其进行身份验证才能访问其内部数据。这是 Kubernetes 的一个重要组成部分，因此证明了为什么我们需要对其进行备份。

要与 etcd 数据存储进行接口交互，我们使用一个名为 `etcdctl` 的工具，它也提供了一个帮助菜单——以防你在考试中遇到困难。输入命令 `etcdctl -h` 以获取我们可以运行的命令列表，用于备份 etcd 数据存储。

注意：在考试中你不需要做这件事，但如果你在家里的实验室里做这个练习（例如，使用友好的 Kubernetes），请运行以下命令来安装 `etcdctl` 命令行工具并将其设置为版本 3：`apt update; apt install -y etcd-client; export ETCDCTL_API=3`。

当版本设置为 3 时，帮助菜单包括 `snapshot save` 命令。此外，如果你使用命令 `etcdctl snapshot save -h` 查看`snapshot save` 的帮助页面，你会看到描述“将 etcd 节点后端快照存储到指定的文件中。”在考试当天忘记命令时，像这样的帮助页面在许多方面都是一项宝贵的资源，这就是为什么我在这里详细说明它的原因。

如前一段（以及第二章）所述，etcd 数据存储有自己的服务器证书，这意味着你必须对其进行身份验证才能访问其内部数据。这意味着我们必须将证书颁发机构（CA）、客户端证书和密钥与我们的备份请求一起传递。幸运的是，我们已经知道所有证书都位于 `/etc/Kubernetes/pki/etcd` 目录中。因此，我们可以在全局选项 `--cacert`、`--cert` 和 `--key`（我们也可以从帮助页面中看到全局选项）的现有位置引用它们。最终的命令是 `etcdctl snapshot save etcdbackup1 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key`。一旦备份完成，我们可以运行命令 `etcdctl snapshot status etcdbackup1 > snapshot-status.txt` 来获取备份的状态，并将其重定向到文件。这些命令的结果将类似于以下内容：

```
root@kind-control-plane:~# etcdctl snapshot save etcdbackup1 --cacert 
➥ /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt 
➥ --key /etc/kubernetes/pki/etcd/server.key
2023-01-30 17:41:32.602411 I | clientv3: opened snapshot stream; 
➥ downloading
2023-01-30 17:41:32.624918 I | clientv3: completed snapshot read; closing
Snapshot saved at etcdbackup1
root@kind-control-plane:~# etcdctl snapshot status etcdbackup1
9ec8949e, 1662, 807, 1.7 MB
root@kind-control-plane:~# etcdctl snapshot status etcdbackup1 > snapshot-
➥ status.txt
```

### D.2.5 恢复 etcd

如果你还没有完成前面的练习，接下来的练习将无法工作，因为它会从你进行备份的点继续。所以，现在我们有了名为`etcdbackup1`的快照，我们可以使用`etcdctl`命令行工具进行恢复，有一个选项是恢复而不是备份。再次，如果我们运行命令`etcdctl snapshot -h`，我们会看到可用的命令是`restore`、`save`和`status`。选择`restore`命令，并运行命令`etcdctl snapshot restore -h`以获取更多信息。从帮助菜单的输出中，看起来我们可以指定一个数据目录。这将允许我们恢复到一个 etcd 已经访问过的目录。我们知道当前 etcd 数据目录的目录是`/var/lib`。我们知道这一点是因为 etcd 的清单，它位于`/etc/kubernetes/manifests`目录中。我们可以运行命令`cat /etc/kubernetes/manifests/etcd.yaml | tail -10`。输出将看起来像这样：

```
root@kind-control-plane:~# cat /etc/kubernetes/manifests/etcd.yaml | 
➥ tail -10
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

这意味着我们可以指定一个类似的目录，比如说`/var/lib/etcd-restore`。完整的命令是`etcdctl snapshot restore snapshotdb --data-dir /var/lib/etcd-restore`。

### D.2.6 升级控制平面

每次你在考试中看到“升级”这个词，就想到 kubeadm。在考试前多次使用 kubeadm 进行升级是一个好习惯。我总是喜欢使用帮助菜单来找到我的方法，所以，让我们再次运行命令`kubeadm -h`来查看我们可用于升级的选项。从输出中，可用的选项中包括`upgrade`命令。如果我们使用命令`kubeadm upgrade -h`在帮助页面中再深入一级，我们可以推断出`plan`将检查我们的集群以选择可用的版本。让我们试试，使用命令`kubeadm upgrade plan`。输出将会非常长，但重要部分看起来像这样：

```
Upgrade to the latest version in the v1.24 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.24.7   v1.24.10
kube-controller-manager   v1.24.7   v1.24.10
kube-scheduler            v1.24.7   v1.24.10
kube-proxy                v1.24.7   v1.24.10
CoreDNS                   v1.8.6    v1.8.6
etcd                      3.5.3-0   3.5.3-0

You can now apply the upgrade by executing the following command:

    kubeadm upgrade apply v1.24.10
```

如果我们比较版本，我们看到我们可以从 v1.24.7 升级到 v1.24.10。输出甚至给出了要运行的精确命令，这非常方便！让我们运行命令`kubeadm upgrade apply v1.24.10`。

注意：你可能会收到错误信息“指定的升级版本‘v1.24.10’高于 kubeadm 版本‘v1.24.7’。”你可以通过运行命令`apt update; apt install -y kubeadm=1.24.10-00.`来修复此信息。

## D.3 第三章考试练习

这些练习是相互关联的，所以我建议你一次性完成第三章的所有考试练习。这将为你准备 CKA 考试提供最佳实践。

### D.3.1 创建角色

要在 Kubernetes 中创建角色，我们使用`kubectl`命令行工具。确保使用帮助菜单，因为通常有一些示例可以直接复制粘贴到你的终端进行考试。例如，如果你运行命令`k create role -h`，你会得到几个示例，如下所示：

```
Examples:
  # Create a role named "pod-reader" that allows user to perform "get", 
➥ "watch" and "list" on pods
  kubectl create role pod-reader --verb=get --verb=list --verb=watch -
➥ resource=pods

  # Create a role named "pod-reader" with ResourceName specified
  kubectl create role pod-reader --verb=get --resource=pods --resource-
➥ name=readablepod --resource-name=anotherpod

  # Create a role named "foo" with API Group specified
  kubectl create role foo --verb=get,list,watch --resource=rs.extensions

  # Create a role named "foo" with SubResource specified
  kubectl create role foo --verb=get,list,watch --resource=pods,pods/status
```

这不是个玩笑；您可以将这些内容复制粘贴到考试时的终端中，我建议您这样做。对于这个练习，因为它将允许为服务账户使用 `create` 动词，我们将从帮助菜单中复制第一个示例，并将其修改为 `kubectl create role sa-creator --verb=create --resource=sa.`。

这将给我们一个新的角色，允许我们创建服务账户。我们可以通过命令 `k get role sa-creator -o yaml` 看到这个角色存在并且具有适当的权限。输出将类似于以下内容：

```
root@kind-control-plane:~# k get role sa-creator -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: "2023-02-17T02:54:46Z"
  name: sa-creator2
  namespace: default
  resourceVersion: "190882"
  uid: 2517a9db-0a1c-4b3b-bf40-59fa799b5fd8
rules:
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
```

### D.3.2 创建角色绑定

以下练习只能在您已经创建了 `sa-creator` 角色的情况下完成。如果您还没有创建，请完成本章的第一个考试练习。与创建角色类似，我们可以使用帮助菜单来找到正确的命令。让我们运行命令 `k create rolebinding -h`，我们将看到一个有用的示例，我们可以根据需要修改它。输出中的示例应类似于以下内容：

```
Examples:
  # Create a role binding for user1, user2, and group1 using the admin 
➥ cluster role
  kubectl create rolebinding admin --clusterrole=admin --user=user1 -
➥ user=user2 --group=group1
```

让我们复制并粘贴这个示例，并将其修改为 `kubectl create rolebinding sa-creator-binding --role=sa-creator --user=sandra`。一旦创建了角色绑定，您可以使用命令 `k get rolebinding sa-creator-binding -o yaml` 来验证设置。输出应类似于以下内容：

```
root@kind-control-plane:~# k get rolebinding sa-creator-binding -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2023-02-17T03:03:28Z"
  name: sa-creator-binding
  namespace: default
  resourceVersion: "191629"
  uid: 0191d224-654b-44fb-8824-2b4e68028fef
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sa-creator
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: sandra
 Using auth can-i 
```

再次，我们将使用帮助菜单来为我们提供正确的命令提示。命令 `k auth can-i -h` 的输出将给出以下示例：

```
Examples:
  # Check to see if I can create pods in any namespace
  kubectl auth can-i create pods --all-namespaces

  # Check to see if I can list deployments in my current namespace
  kubectl auth can-i list deployments.apps

  # Check to see if I can do everything in my current namespace ("*" means 
➥ all)
  kubectl auth can-i '*' '*'

  # Check to see if I can get the job named "bar" in namespace "foo"
  kubectl auth can-i list jobs.batch/bar -n foo

  # Check to see if I can read pod logs
  kubectl auth can-i get pods --subresource=log

  # Check to see if I can access the URL /logs/
  kubectl auth can-i get /logs/

  # List all allowed actions in namespace "foo"
  kubectl auth can-i --list --namespace=foo
```

我们可以推断出要运行的命令是 `kubectl auth can-i create sa --as sandra`。运行该命令的结果将类似于以下内容：

```
root@kind-control-plane:~# k auth can-i create sa --as sandra
yes
```

### D.3.3 创建新用户

通过阅读第三章，您应该明白用户只是一个构造，而不是用户数据库中的实际用户。这意味着尽管我们创建了用户 Sandra，我们只是在创建一个证书，其中通用名称是 Sandra。Kubernetes 没有用户这一概念，但可以与其他身份提供者集成。对于考试，您需要知道如何生成这个证书，所以我建议您多次进行这个练习，因为 Kubernetes 文档中（您可以在考试期间打开）没有直接的答案。让我们首先使用 `openssl` 命令行工具，这个工具将在考试中可用（已安装），并且所有 Linux 系统都自带，包括您用于练习实验室的系统。让我们使用命令 `openssl genrsa -out sandra.key 2048.` 生成一个 2048 位加密的私钥。输出将类似于以下内容：

```
root@kind-control-plane:/# openssl genrsa -out sandra.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
......................................................+++++
.......................+++++
e is 65537 (0x010001)
```

现在，让我们使用我们刚刚创建的私钥制作一个证书签名请求文件，我们最终会将它提供给 Kubernetes API。在这里，我们指定用户 Sandra 为证书签名请求的通用名称非常重要，使用命令 `openssl req -new -key carol.key -subj "/CN=sandra" -out sandra.csr`：

```
root@kind-control-plane:/# openssl req -new -key carol.key -subj 
➥ "/CN=carol/O=developers" -out carol.csr
root@kind-control-plane:/# ls | grep carol
carol.csr
carol.key
```

接下来，将 CSR 文件存储在环境变量中，因为我们稍后会需要它。为此，使用命令`export REQUEST=$(cat sandra.csr | base64 -w 0)`将 CSR 文件的 Base64 编码版本存储在名为`REQUEST`的环境变量中。

然后，使用以下多行命令从该请求创建 CSR 资源：

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: sandra
spec:
  groups:
  - developers
  request: $REQUEST
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```

这将在一个命令中创建资源并输入请求。你可以使用命令`k get csr`查看请求。输出将看起来像这样：

```
root@kind-control-plane:/# kubectl get csr
NAME    AGE   SIGNERNAME                            REQUESTOR          
➥ CONDITION
sandra  4s    kubernetes.io/kube-apiserver-client   kubernetes-admin   
➥ Pending
```

你可以使用命令`kubectl certificate approve Sandra`批准请求，你会看到条件从`pending`变为`Approved,Issued`：

```
root@kind-control-plane:/# kubectl certificate approve sandra
certificatesigningrequest.certificates.k8s.io/sandra approved
root@kind-control-plane:/# kubectl get csr
NAME    AGE     SIGNERNAME                            REQUESTOR          
➥ CONDITION
sandra  2m12s   kubernetes.io/kube-apiserver-client   kubernetes-admin   
➥ Approved,Issued
```

现在它已经被批准，你可以从已签名的证书中提取客户端证书，使用 Base64 解码，并使用命令`kubectl get csr sandra -o jsonpath='{.status.certificate}' | base64 -d > carol.crt`将其存储在名为`sandra.crt`的文件中：

```
root@kind-control-plane:/# kubectl get csr sandra -o 
➥ jsonpath='{.status.certificate}' | base64 -d > sandra.crt
```

### D.3.4 将 Sandra 添加到 kubeconfig

如果你想要承担 Sandra 的角色，你必须将用户（以及证书，这是重要部分）添加到 kubeconfig 文件中（我们使用该文件来运行`kubectl`）。要将用户 Sandra 添加到我们的上下文中，我们将运行命令`kubectl config set-context carol --user=sandra --cluster=kind`。一旦运行此命令，你会注意到上下文已经通过运行命令`kubectl config get-contexts`添加。你会看到类似以下的输出：

```
root@kind-control-plane:/# kubectl config set-context sandra --user=sandra 
➥ --cluster=kind
Context "carol" created.
root@kind-control-plane:/# kubectl config get-contexts
CURRENT   NAME                    CLUSTER   AUTHINFO           NAMESPACE
          sandra                  kind      sandra
*         kubernetes-admin@kind   kind      kubernetes-admin
```

左侧`current`列中的星号表示我们目前正在使用哪个上下文。因此，要切换上下文，请运行命令`kubectl config use-context sandra`。你会注意到星号将当前上下文更改为`Sandra`。

### D.3.5 创建新的服务账户

如我们从第三章阅读中得知，服务账户的令牌会自动挂载到 Pod 上，要防止该令牌的挂载需要特殊的配置设置。第一步是查看帮助菜单。我总是从帮助菜单开始，因为在紧张的时刻（考试中），你有时会失去思路，也许会忘记使用哪个命令或命令和选项的顺序。这种情况发生在我们所有人身上！别担心，记住，如果你需要，帮助菜单就在那里。它有存在的理由。例如，你可以运行命令`k create -h`来查看可用命令的列表，你会发现`serviceaccount`被列为可用命令之一。让我们运行命令`k create serviceaccount -h`来查看特定于服务账户的选项。输出将给我们提供很多选项，但最值得注意的是，它将展示以下示例：

```
Examples:
  # Create a new service account named my-service-account
  kubectl create serviceaccount my-service-account
```

我们可以复制并粘贴这个示例，并将其更改为 `kubectl create serviceaccount secure-sa,`，这将创建我们在这个练习中需要的 Service Account。然后，为了确保令牌不会暴露给 Pod，我们可以运行命令 `k get sa secure-sa -o yaml > secure-sa.yaml`，然后运行命令 `echo "automountServiceAccountToken: false" >> secure-sa.yaml` 来确保令牌不会自动挂载到 Pod。要应用更改，运行命令 `k apply -f secure-sa.yaml`，这将应用配置更改以禁用为使用此 Service Account 的所有 Pod 自动挂载令牌。

### D.3.6 创建新的集群角色

我们在本章的第一个练习中创建了一个角色；现在我们将以类似的方式创建一个集群角色。我希望你现在已经熟悉了使用帮助菜单来查找创建集群角色的 `kubectl` 命令的优秀示例。我将从命令 `k create clusterrole -h` 中的第一个示例开始，并将其更改为 `kubectl create clusterrole acme-corp-role --verb=create --resource=deploy,rs,ds`。接下来，我们将创建一个角色绑定（注意它没有提到集群角色绑定），我们将称之为 `acme-corp-role-binding`，将其绑定到 `secure-sa` Service Account，并确保 Service Account 只能在默认命名空间中创建 Deployments、ReplicaSets 和 DaemonSets（而不是 `kube-system` 命名空间）。我们将使用命令 `k create rolebinding -h` 中的示例，并将其更改为 `kubectl create rolebinding acme-corp-role-binding --clusterrole=acme-corp-role --serviceaccount=default:secure-sa`。然后，我们将使用命令 `kubectl -n kube-system auth can-i create deploy --as system:serviceaccount:default:nomount-sa` 来检查角色。你应该得到响应 `no`。

## D.4 第四章考试练习

第四章考试练习围绕在 Kubernetes 中调度 Deployments 或 Pods。你不需要为考试部署有状态集，因为它没有列为考试标准的对象。当你想到调度时，只需想到创建一个 Pod——无论是否在 Deployment 中——并将其放置在节点上。在本章中，有方法可以控制 Pod 放置在哪个节点上，这就是我们将关注的练习内容。

### D.4.1 应用标签并创建 Pod

这个练习仅仅涉及给一个节点应用一个标签。如果你不知道从哪里开始，有一个帮助菜单可以提供帮助！让我们运行命令 `kubectl -h | grep label` 来查看是否有 `label` 在可用的命令中。输出将类似于以下内容：

```
root@kind-control-plane:~# k -h | grep label
  delete          Delete resources by file names, stdin, resources and 
➥ names, or by resources and label selector
  label           Update the labels on a resource 
```

要给节点 `kind-worker` 标签，我们可以运行命令 `k label no kind-worker disk=ssd`。然后，我们可以使用命令 `k get no -show-labels` 来显示我们节点上的标签。输出将类似于以下内容：

```
root@kind-control-plane:~# k get no --show-labels
NAME                 STATUS   ROLES           AGE   VERSION   LABELS
kind-control-plane   Ready    control-plane   8d    v1.24.7   
➥ beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress-
➥ ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-
➥ control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-
➥ plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>          8d    v1.24.7   
➥ beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ssd,kube
➥ rnetes.io/arch=amd64,kubernetes.io/hostname=kind-
➥ worker,kubernetes.io/os=linux
```

我们可以看到标签已成功应用，现在我们可以使用命令`k run fast --image nginx -dry-run=client -o yaml > fast.yaml`创建 Pod 的 YAML。让我们打开文件并更改两行，这将使 Pod 调度到带有标签`disk=ssd`的节点。在文件的最后，就在单词`status`上方，与`restartPolicy`对齐，我们将添加以下行：

```
nodeSelector:
  disk: ssd
```

保存你的更改并关闭文件。现在你有了 YAML 文件，你可以使用命令`k create -f fast.yaml`创建 Pod。确保它被调度到正确的节点，使用命令`k get po -o wide`。输出将类似于以下内容：

```
root@kind-control-plane:~# k get po -o wide
NAME                        READY   STATUS    RESTARTS      AGE   
➥ IP               NODE          NOMINATED NODE   READINESS GATES
fast                         1/1     Running   1 (36h ago)   8d    
➥ 10.244.162.144   kind-worker   <none>           <none>
```

### D.4.2 编辑正在运行的 Pod

当你使用`k edit po`命令编辑正在运行的 Pod 时，你只能更改 YAML 中的某些字段。这是正常行为，即使你无法直接更改，Kubernetes 也会在`/tmp/`目录下的一个文件中保存 Pod YAML 的副本。让我们运行命令`k edit po fast`来查看它看起来像什么，这将打开 Vim 编辑器中的 Pod YAML。你可以通过按键盘上的斜杠(/)键并跟随着单词`nodeSelector`（例如，在 Vim 中输入`/nodeSelector`）然后按 Enter 键来搜索`nodeSelector`。Vim 文本编辑器将首先突出显示注释中的单词`nodeSelector`，因此按 N 键进入下一个结果，这将是我们正在寻找的结果。然后你可以按键盘上的 I 键进入插入模式。将文本从`disk: ssd`更改为`disk: slow`。当你完成这个练习时，必须更改的 YAML 部分将看起来像以下内容：

```
  nodeName: kind-worker
  nodeSelector:
    disk: slow
  preemptionPolicy: PreemptLowerPriority
```

现在你已经更改了 YAML 文件，你可以保存并退出文件；然而，你会收到一个警告信息，指出 Pod 更新可能不会更改除`spec.containers[*].image`之外的字段。这没关系，因为你将继续退出文件。只有在这种情况下，文件才会被保存到`/tmp`目录。当你退出文件时，你收到的信息将类似于以下内容：

```
root@kind-control-plane:~# k edit po fast
error: pods "fast" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-
➥ 589741394.yaml"
error: Edit cancelled, no valid changes were saved.
```

即使收到了错误，这也还是可以的。这是我们将存储在`/tmp`目录中的 YAML 应用的部分，并强制 Pod 重新启动。执行此操作的命令是`k replace -f /tmp/kubectl-edit-589741394.yaml --force`（YAML 文件的名称将因你而异）。这将导致当前运行的 Pod 被删除，并创建一个新的 Pod（具有新的名称）。输出将类似于以下内容：

```
root@kind-control-plane:~# k replace -f /tmp/kubectl-edit-589741394.yaml -
➥ force
pod "fast" deleted
pod/fast replaced
```

一旦完成，你可以使用命令`k get po fast; k get po fast -o yaml | grep disk.`来检查 Pod 是否使用新的配置正在运行。

```
root@kind-control-plane:~# k get po fast; k get po fast -o yaml | grep disk:
NAME   READY   STATUS         RESTARTS   AGE
fast   1/1     Running        0          4m39s
    disk: slow
```

### D.4.3 为新 Pod 使用节点亲和性

如我们从第四章阅读中了解到的，节点亲和性是根据对具有特定标签的节点的一定偏好来调度 Pod。例如，在本练习中，我们表示 Pod 应该被调度到具有标签 `disk=ssd` 作为首选的节点。然而，有一个备用计划，以防没有具有 `disk=ssd` 标签的可用节点。备用计划是将 Pod 调度到具有标签 `Kubernetes.io/os=linux` 的节点。

让我们从使用命令 `k run ssd-pod -image nginx -dry-run=client -o yaml > ssd-pod.yaml` 创建 Pod 的 YAML 开始。现在打开文件 `ssd-pod.yaml` 并插入节点亲和性配置。对于考试，我会在 Kubernetes 文档中利用这一点，您可以在考试期间打开它。因此，在网页浏览器中打开网站 [`kubernetes.io/docs`](https://kubernetes.io/docs)，在搜索栏中输入 `node selector` 并按 Enter。选择第一个链接，命名为 Assigning Pods to Nodes，然后点击页面右侧的链接 Affinity and Anti-affinity。您将看到此页面上列出的 YAML，您可以将其复制并粘贴到现有的 `ssd-pod.yaml` 文件中。您需要复制的是 `spec:` 下的整个亲和性部分。我们将对其进行轻微修改，使其看起来如下：

```
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
```

一旦将内容粘贴到文件 `ssd-pod.yaml` 中，您就可以保存并退出。您可以使用命令 `k apply -f ssd-pod.yaml` 创建 Pod。使用命令 `k get po -o wide` 检查 Pod 是否已成功调度到正确的节点。输出应类似于以下内容：

```
root@kind-control-plane:~# k apply -f ssd-pod.yaml
pod/ssd-pod created
root@kind-control-plane:~# k get po -o wide
NAME                        READY   STATUS    RESTARTS      AGE   
➥ IP               NODE          NOMINATED NODE   READINESS GATES
ssd-pod                     1/1     Running   0             16s   
➥ 10.244.162.152   kind-worker   <none>           <none>
```

## D.5 第五章考试练习

这些练习与上一组练习不同，因为它们更多地与当前运行的 Deployments 和 Pods 的维护有关。这些练习将涉及扩展、更新镜像和查看 Deployment 的滚动更新。这对于考试很有帮助，因为您可能需要将应用程序部署到 Kubernetes 的新版本，或者可能需要回滚到以前的版本。

### D.5.1 在 Deployment 中扩展副本

对于考试来说，很可能已经有一个 Deployment 在运行，但为了在我们的实践实验室中模拟这种情况，我们不得不自己创建一个。这个练习最重要的部分是扩展操作，而不是创建 Deployment。让我们先运行命令 `k create deploy apache --image httpd:latest`。这将创建 Deployment，并且由于我们没有在命令中指定，Deployment 将只有一个副本。您可以使用命令 `k get deploy,po` 验证 Deployment 内的 Pod 是否已创建。输出将类似于以下内容：

```
root@kind-control-plane:~# k get deploy,po
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/apache     1/1     1            1           3m30s

NAME                            READY   STATUS    RESTARTS      AGE
pod/apache-67984dc457-5mcvj     1/1     Running   0             3m30s
```

现在 Pod 已经启动并运行，我们可以将 Deployment 从单个副本扩展到五个副本。扩展副本的命令是 `k scale deploy apache -replicas 5`。一旦我们运行该命令，我们将看到另外四个 Pod 被启动。为了验证这是否发生，再次运行命令 `k get deploy,po`。输出现在将看起来像这样：

```
root@kind-control-plane:~# k scale deploy apache --replicas=5
deployment.apps/apache scaled
root@kind-control-plane:~# k get deploy,po
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/apache     3/5     5            3           5m57s

NAME                            READY   STATUS              RESTARTS      
➥ AGE
pod/apache-67984dc457-5mcvj     1/1     Running             0             
➥ 5m57s
pod/apache-67984dc457-bcs6q     1/1     Running             0             
➥ 3s
pod/apache-67984dc457-dwzl9     0/1     ContainerCreating   0             
➥ 3s
pod/apache-67984dc457-kl7rq     0/1     ContainerCreating   0             
➥ 3s
pod/apache-67984dc457-rdgh5     1/1     Running             0             
➥ 3s
```

### D.5.2 更新镜像

在上一个练习中，我们创建了一个名为 `apache` 的 Deployment，我们将在本练习中继续使用相同的 Deployment。如果你还没有完成上一个练习，请在开始这个练习之前完成它。正如我们在第五章中了解到的，在 Deployment 中更新镜像会导致 Kubernetes 创建一个新的 ReplicaSet，并将此操作记录为新的滚动。我们可以使用命令 `k set image deploy apache httpd=httpd:latest httpd=httpd:2.4.54` 来更新镜像。你可以使用命令 `k get deploy apache -o yaml | grep image` 验证 Deployment 是否包含正确的镜像。输出将看起来像这样：

```
root@kind-control-plane:~# k get deploy apache -o yaml | grep image
      - image: httpd:2.4.54
        imagePullPolicy: Always
```

### D.5.3 查看 ReplicaSet 事件

每次你更改镜像，以及部署的其他特性，都会创建一个新的 ReplicaSet。这是因为 Pods 的配置发生了变化；因此，旧的 Pods 会被终止，新的 Pods 会被创建。正如我们在第五章中了解到的，这一切都是由 ReplicaSet 处理的，它是控制器，用于帮助部署新版本的 Deployment。我们可以通过运行命令 `k describe rs apache-67984dc457` 来查看事件。请注意，ReplicaSet 的名称对于你来说可能会有所不同。输出包含大量信息，但重要的是在末尾的事件部分，它应该看起来像这样：

```
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  58m    replicaset-controller  Created pod: 
➥ apache-67984dc457-kl7rq
  Normal  SuccessfulCreate  58m    replicaset-controller  Created pod: 
➥ apache-67984dc457-rdgh5
  Normal  SuccessfulCreate  58m    replicaset-controller  Created pod: 
➥ apache-67984dc457-dwzl9
  Normal  SuccessfulCreate  58m    replicaset-controller  Created pod: 
➥ apache-67984dc457-bcs6q
  Normal  SuccessfulDelete  7m59s  replicaset-controller  Deleted pod: 
➥ apache-67984dc457-rdgh5
  Normal  SuccessfulDelete  6m45s  replicaset-controller  Deleted pod: 
➥ apache-67984dc457-bcs6q
  Normal  SuccessfulDelete  6m44s  replicaset-controller  Deleted pod: 
➥ apache-67984dc457-5mcvj
  Normal  SuccessfulDelete  6m43s  replicaset-controller  Deleted pod: 
➥ apache-67984dc457-dwzl9
  Normal  SuccessfulDelete  6m43s  replicaset-controller  Deleted pod: 
➥ apache-67984dc457-kl7rq
```

### D.5.4 回滚到之前的应用程序版本

现在我们已经成功推出到新版本（通过更改部署镜像），我们可以通过执行命令 `k rollout undo deploy apache` 轻松回滚到上一个版本，使用之前的镜像。现在，通过运行命令 `k rollout status deploy apache`，然后 `k rollout history deploy apache`，我们可以看到我们当前处于哪个版本。

```
root@kind-control-plane:~# k rollout undo deploy apache
deployment.apps/apache rolled back
root@kind-control-plane:~# k rollout status deploy apache
Waiting for deployment "apache" rollout to finish: 1 old replicas are 
➥ pending termination...
Waiting for deployment "apache" rollout to finish: 1 old replicas are 
➥ pending termination...
Waiting for deployment "apache" rollout to finish: 1 old replicas are 
➥ pending termination...
Waiting for deployment "apache" rollout to finish: 4 of 5 updated replicas 
➥ are available...
deployment "apache" successfully rolled out
root@kind-control-plane:~# k rollout history deploy apache
deployment.apps/apache
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

### D.5.5 更改滚动策略

要更改滚动策略，我们必须修改 Deployment YAML。我们可以通过运行命令 `k edit deploy apache` 来轻松完成此操作。这将打开 Vim 文本编辑器中的 Deployment YAML。我们可以找到以 `strategy` 开头的行，并将类型更改为 `Recreate`。策略下的其余 YAML 可以删除。最终结果将看起来像这样：

```
      app: apache
  strategy:
     type: Recreate
  template:
```

一旦你将策略从 `RollingUpdate` 更改为 `Recreate`，你可以保存并退出文件。Deployment 将自动更新。不会发生滚动，因为这只会影响下一个滚动阶段。退出 Deployment YAML 后的结果将看起来像这样：

```
root@kind-control-plane:~# k edit deploy apache
deployment.apps/apache edited
```

作为额外加分，你可以再次执行之前的练习，并看到在创建新 Pods 之前，所有 Pods 都已被终止，因为这正是我们打算更改的部署策略。

### D.5.6 隔离和解除隔离节点

对于这个练习，我们首先创建一个三节点集群。请参阅附录 A 了解如何使用 kind Kubernetes 创建多节点集群。当三节点集群启动并运行，并且你已经运行了命令 `docker exec -it kind-control-plane` 以获取对控制平面节点（其中已安装 `kubectl`）的访问权限后，你可以继续这个任务。

隔离一个节点意味着我们将它标记为不可调度。这并不一定意味着从节点中驱逐 Pods。你必须运行 drain 命令来完成这个操作。要隔离一个节点，请运行命令 `k cordon kind-worker`。你可以通过运行命令 `k get no` 来验证此节点上的调度已被禁用。输出将类似于以下内容：

```
root@kind-control-plane:/# k cordon kind-worker
node/kind-worker cordoned
root@kind-control-plane:/# k get no
NAME                 STATUS                     ROLES           AGE   
➥ VERSION
kind-control-plane   Ready                      control-plane   58s   
➥ v1.26.0
kind-worker          Ready,SchedulingDisabled   <none>          38s   
➥ v1.26.0
kind-worker2         Ready                      <none>          38s   
➥ v1.26.0
```

现在，我们可以调度一个 Pod，并且它应该被调度到节点 `kind-worker2`，因为我们刚刚隔离了节点 `kind-worker`。要调度一个 Pod，让我们运行命令 `k run nginx --image nginx`。然后，我们可以使用命令 `k get po -o wide` 检查 Pod 是否被调度到正确的节点。输出应该类似于以下内容：

```
root@kind-control-plane:/# k run nginx --image nginx
pod/nginx created
root@kind-control-plane:/# k get po -o wide
NAME    READY   STATUS              RESTARTS   AGE   IP       NODE         
➥ NOMINATED NODE   READINESS GATES
nginx   0/1     ContainerCreating   0          5s    <none>   kind-worker2 
➥ <none>           <none>
```

接下来，我们将使用命令 `k uncordon kind-worker` 解除节点 `kind-worker` 的隔离状态。现在节点已解除隔离，我们可以再次向其调度 Pods。请通过添加节点选择器将 Pod 从节点 `kind-worker2` 移动到节点 `kind-worker`。我们将在 Pod YAML 中使用命令 `k edit po nginx` 添加节点选择器（你可能需要运行命令 `apt update; apt install -y vim` 重新在你的新 kind 集群上安装 Vim）。这将使用 Vim 文本编辑器打开 YAML 文件。在文件中找到以 `nodeName` 开头的行（第 29 行）。将此行从 `nodeName: kind-worker2` 更改为 `nodeName: kind-worker`。保存并退出文件。这将提示你一个消息，表明 Pod 更新可能不会更改除 `spec.containers[*].image` 之外的字段，这是正常的，因为你将继续退出文件。一旦你退出文件，你会注意到它在 `/tmp` 目录中存储了一个 YAML 的副本。你可以运行命令 `k replace -f /tmp/kubectl-edit-3840075995.yaml --force` 来终止旧的 nginx Pod，这将创建一个新的 Pod 并将其调度到节点 `kind-worker`。我们可以通过运行命令 `k get po -o wide` 来验证这一点。

```
root@kind-control-plane:/# k edit po nginx
error: pods "nginx" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-
➥ 3840075995.yaml"
error: Edit cancelled, no valid changes were saved.
root@kind-control-plane:/# k replace -f /tmp/kubectl-edit-3840075995.yaml 
➥ --force
pod "nginx" deleted
pod/nginx replaced
root@kind-control-plane:/# k get po -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP           NODE          
➥ NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          4m17s   10.244.1.2   kind-worker   
➥ <none>           <none>
```

我们可以看到，实际上它被调度到了节点 `k``ind-worker`。

### D.5.7 从节点中移除污点

考试可能已经有一个 Deployment 在运行，但为了你的实践实验室环境，你可能没有，所以我们将首先创建一个 Deployment 来模拟这种考试场景。我们可以使用之前练习中使用的相同集群。

我们首先使用命令 `k create deploy nginx -image nginx` 创建一个名为 nginx 的 Deployment，使用 nginx 镜像。然后，我们将使用命令 `k taint no kind-control-plane node-role.kubernetes.io/control-plane:NoSchedule-` 从控制平面节点移除污点。现在污点已经被移除，我们可以将 Pod 调度到该节点，无需包含对污点的容忍。让我们继续这样做，但首先，我们需要修改 Deployment YAML。我们可以使用命令 `k edit deploy nginx``,` 来打开 Vim 文本编辑器中的 Deployment YAML。我们可以在 Pod 规范中添加一个节点选择器，通过插入以下内容：

```
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/hostname: kind-control-plane
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

一旦我们添加了节点选择器行，新的配置将自动应用，因为 Deployment 控制器将识别到变化并重新调度 Pod 到控制平面节点。我们可以使用命令 `k get po -o wide` 验证这一点。输出将类似于以下内容：

```
root@kind-control-plane:/# k edit deploy nginx
deployment.apps/nginx edited
root@kind-control-plane:/# k get po -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP           
➥ NODE                 NOMINATED NODE   READINESS GATES
nginx                   1/1     Running   0          27m     10.244.1.2   
➥ kind-worker          <none>           <none>
nginx-cd5574b4f-qbz9z   1/1     Running   0          3m46s   10.244.0.5   
➥ kind-control-plane   <none>           <none>
```

## D.6 第六章考试练习

第六章的考试练习变得更加复杂，因为我们开始处理集群内部的 DNS 和通信。这些练习是相互关联的，所以我建议您从第一个开始，然后继续第二个、第三个等等。如果您试图在练习的中间开始，您将无法继续进行，除非您已经完成了前面的练习。我们将在这里详细说明如何解决这些练习，以便您为考试做好最佳准备。

### D.6.1 将执行进入 Pod

对于考试，您已经有了已经创建的 Pod，但为了在您自己的个人实验室环境中（例如，kind Kubernetes）进行练习，请继续创建 Pod 作为先决条件。如果您是从第五章的练习继续，您可以使用我们创建的名为 `nginx` 的 Pod。如果您是从零开始，您可以使用命令 `k run nginx -image nginx` 创建一个 Pod。一旦创建了 Pod，您可以使用容器内的命令检查它是否正在运行。为了获取 Pod 用于解析域名（在运行时注入到每个 Pod 中）的 DNS 服务器的 IP 地址，我们可以运行命令 `k exec -it nginx --cat /etc/resolv.conf`。命令的输出将类似于以下内容：

```
root@kind-control-plane:/# k exec -it nginx --cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

### D.6.2 更改 DNS 服务

要更改 DNS 服务 IP 地址，首先更改 API 服务器 YAML 配置中的 CIDR 范围。我们可以通过更改位于 `/etc/kubernetes/manifests` 的 YAML 文件来完成，该文件名为 `kube-apiserver.yaml`。让我们打开它并修改 YAML 中的 `service-cluster-ip` 值。我们将运行命令 `vim /etc/kubernetes/manifests/kube-apiserver.yaml`，这将使用 Vim 文本编辑器打开文件，然后我们可以将 CIDR 从 `10.96.0.0/6` 更改为 `100.96.0.0/6`。一旦我们做出更改，我们可以保存并退出，更改将自动生效。你可能需要等待最多 5 分钟，以便在 kube-system 命名空间中重新创建 API 服务器 Pod。现在，找到 DNS 服务，它始终位于 kube-system 命名空间中。让我们运行命令 `k -n kube-system get svc` 来查看服务。命令的输出将类似于以下内容：

```
root@kind-control-plane:/# k -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  
➥ AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   
➥ 107m
```

集群 IP 是 10.96.0.10，在这个练习中，我们将将其更改为 100.96.0.10。我们可以使用命令 `k -n kube-system edit svc kube-dns` 来更改服务 IP 地址，这将在一个 Vim 文本编辑器中打开服务的 YAML。一旦 YAML 打开，我们可以更改位于 spec 下方两个值。我们将把该 IP 地址的两个实例从 `10.96.0.10` 更改为 `100.96.0.10`。一旦我们做出更改，我们可以保存并退出。我们可能会看到警告信息，表明你无法更改此值。你可以继续退出文件 (!q)，并且 YAML 文件的副本将被存储在 `/tmp` 目录中。输出将类似于以下内容：

```
root@kind-control-plane:/# k -n kube-system edit svc kube-dns
error: services "kube-dns" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-
➥ 2356510614.yaml"
error: Edit cancelled, no valid changes were saved.
```

让我们运行一个 YAML 的替换，这将终止服务并创建一个新的服务，其中包含我们为 DNS 的新 IP 地址。为此，我们将运行命令 `k replace -f /tmp/kubectl-edit-2356510614.yaml --force` 来替换服务的实例。

### D.6.3 修改 kubelet 配置

现在我们已经修改了 kube-dns 服务，我们需要修改 Pod 的 kubelet 配置以获取新的 DNS IP 信息。这可以通过修改 `/var/lib/kubelet/` 目录下名为 `config.yaml` 的文件来完成。让我们使用命令 `vim /var/lib/kubelet/config.yaml` 打开该文件。一旦文件在 Vim 中打开，我们可以将 `clusterDNS` 的值从 `10.96.0.10` 更改为 `100.96.0.10`。完成此操作后，你可以保存并退出文件。现在，我们已经更改了服务配置，我们需要使用命令 `systemctl daemon-reload` 和 `systemctl restart kubelet` 重新加载 kubelet 守护进程，以在节点上重启 kubelet 服务。最后，我们将使用命令 `systemctl status kubelet` 验证服务是否处于活动状态并正在运行。输出将包含大量信息，但重要的是服务处于活动状态并正在运行，这可以在输出的开头找到：

```
root@kind-control-plane:/# systemctl status kubelet
   kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor 
➥ preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sat 2023-02-18 19:04:20 UTC; 1min 20s 
➥ ago
       Docs: http://kubernetes.io/docs/
```

### D.6.4 编辑 kubelet ConfigMap

要定位 ConfigMap，它类似于 Service，位于 kube-system 命名空间中，我们可以运行命令 `k -n kube-system get cm`。输出将类似于以下内容：

```
root@kind-control-plane:/# k -n kube-system get cm
NAME                                 DATA   AGE
coredns                              1      134m
extension-apiserver-authentication   6      134m
kube-proxy                           2      134m
kube-root-ca.crt                     1      133m
kubeadm-config                       1      134m
kubelet-config                       1      134m
```

我们在这个练习中特别关注的 ConfigMap 被命名为 `kubelet-config`。我们可以使用命令 `k -n kube-system edit cm kubelet-config` 来编辑 ConfigMap。这将在一个 Vim 文本编辑器中打开 ConfigMap 的 YAML 文件。向下滚动到以 `clusterDNS` 开头的行，并将下面的值从 `10.96.0.10` 更改为 `100.96.0.10`。我们可以保存并退出文件以自动应用更改。输出将类似于以下内容：

```
root@kind-control-plane:/# k -n kube-system edit cm kubelet-config
configmap/kubelet-config edited
```

现在我们已经升级了节点，因为 kubelet 是当前在节点上运行的守护进程，我们必须更新此配置的节点，以及重新加载守护进程并在节点上重新启动 kubelet 服务。首先，为了更新节点上的 kubelet 配置，执行命令 `kubeadm upgrade node phase kubelet-config`。输出将类似于以下内容：

```
root@kind-control-plane:/# kubeadm upgrade node phase kubelet-config
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-
➥ system get cm kubeadm-config -o yaml'
W0914 17:44:33.203828    3618 utils.go:69] The recommended value for 
➥ "clusterDNS" in "KubeletConfiguration" is: [10.96.0.10]; the provided 
➥ value is: [100.96.0.10]
[kubelet-start] Writing kubelet configuration to file 
➥ "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using 
➥ your package manager.
```

现在我们已经升级了节点的 kubelet 配置，我们可以使用命令 `systemctl daemon-reload` 重新加载守护进程，并使用命令 `systemctl restart kubelet` 重新启动服务。您将不会收到输出；您将直接返回到命令提示符，所以只要没有错误消息，您已成功重新启动 kubelet 服务。

### D.6.5 缩放 CoreDNS 部署

Kubernetes 中的 DNS 服务作为 Deployment 运行。为了缩放此 Deployment，我们将运行用于缩放任何其他 Deployment 的相同命令，即 `k -n kube-system scale deploy Coredns -replicas 3`。然后，您可以使用命令 `k -n kube-system` 检查副本是否已缩放。输出应类似于以下内容：

```
root@kind-control-plane:/# k -n kube-system scale deploy coredns --replicas 
➥ 3
deployment.apps/coredns scaled
root@kind-control-plane:/# k -n kube-system get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   3/3     3            3           141m
```

### D.6.6 从 Pod 验证 DNS 更改

现在我们已经对 DNS 进行了这些更改，我们将验证新创建的 Pod 是否也接收到了这些更改。创建名为 `netshoot` 的 Pod 并使用 netshoot 镜像的命令是 `kubectl run netshoot --image=nicolaka/netshoot --command sleep --command "3600"`。我们运行 `sleep` 和 `3600` 这两个命令，以便 Pod 保持运行状态（3,600 秒，或 60 分钟）。我们可以使用命令 `k get po` 检查 Pod 是否处于运行状态。输出应类似于以下内容：

```
root@kind-control-plane:/# kubectl run netshoot --image=nicolaka/netshoot -
➥ -command sleep --command "3600"
pod/netshoot created
root@kind-control-plane:/# k get po
NAME       READY   STATUS    RESTARTS   AGE
netshoot   1/1     Running   0          40s 
```

Pod 正在运行，因此现在您可以通过容器获取 Bash shell。为此，执行命令 `k exec -it netshoot -bash`。您会注意到您的提示符已更改，这意味着您已成功进入 Pod 内的容器。输出应类似于以下内容：

```
root@kind-control-plane:/# k exec -it netshoot --bash
bash-5.1#
```

现在您已经在容器中打开了 Bash shell，您可以使用命令 `cat /etc/resolv.conf` 来检查是否列出了正确的 DNS IP 地址。输出应类似于以下内容：

```
root@kind-control-plane:/# k exec -it netshoot --bash
bash-5.1# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 100.96.0.10
options ndots:5 
```

这意味着 DNS 已正确配置；因此，Pod 能够在集群中使用 CoreDNS 解析 DNS 名称。你可以使用命令 `nslookup example.com` 检查这个 Pod 是否能够解析到 example.com 的 DNS 查询。Nslookup 是一个 DNS 工具，允许你查询名称服务器。输出应如下所示：

```
bash-5.1# nslookup example.com
Server:    100.96.0.10
Address:    100.96.0.10#53

Non-authoritative answer:
Name:    example.com
Address: 93.184.216.34
Name:    example.com
Address: 2606:2800:220:1:248:1893:25c8:1946
```

### D.6.7 创建 Deployment 和 Service

要使用镜像 `nginxdemos/hello:plain-text` 创建名为 `hello` 的 Deployment，可以运行命令 `k create deploy hello --image nginxdemos/hello:plain-text`。然后，我们可以使用命令 `k expose deploy hello --name hello-svc --port 80` 来暴露 Deployment。为了验证我们是否正确创建了服务，我们将运行命令 `k get svc`。前述命令的输出将如下所示：

```
root@kind-control-plane:/# k create deploy hello --image 
➥ nginxdemos/hello:plain-text
deployment.apps/hello created
root@kind-control-plane:/# k expose deploy hello --name hello-svc --port 80
service/hello-svc exposed
root@kind-control-plane:/# k get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hello-svc    ClusterIP   100.96.226.92   <none>        80/TCP    2s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   153m
```

### D.6.8 将 ClusterIP 服务改为 NodePort

要更改服务类型，我们可以运行命令 `k edit svc hello-svc`，这将打开 YAML 文件在 Vim 文本编辑器中。在 spec 下，我们可以将起始行包含 `type` 的行（第 32 行）从 `type: ClusterIP` 改为 `type: NodePort`。然后，我们可以在端口列表下添加 `nodePort: 3000`（直接位于 `port: 80` 之下）。更改后的最终 YAML 将如下所示：

```
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30000
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hello
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

保存并退出文件后，更改将自动应用。我们可以使用命令 `k get svc` 再次查看服务。输出将如下所示：

```
root@kind-control-plane:/# k get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-svc    NodePort    100.96.226.92   <none>        80:30000/TCP   5m5s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        159m
```

要通过节点端口通过服务访问应用程序，我们必须获取 Pod 的位置（它在哪个节点上）和节点的 IP 地址。我们可以使用命令 `k get po -o wide; k get no -o wide` 来查看这些信息。输出将如下所示：

```
root@kind-control-plane:/# k get po -o wide; k get no -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP           
➥ NODE                 NOMINATED NODE   READINESS GATES
hello-5dc6ddf4c4-qrwq4   1/1     Running   0          12m    10.244.2.7   
➥ kind-worker2         <none>           <none>
nginx                    1/1     Running   0          62m    10.244.1.3   
➥ kind-worker          <none>           <none>
nginx-cd5574b4f-qbz9z    1/1     Running   0          130m   10.244.0.5   
➥ kind-control-plane   <none>           <none>
NAME                 STATUS   ROLES           AGE    VERSION   INTERNAL-IP   
➥ EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   166m   v1.26.0   172.18.0.7    
➥ <none>        Ubuntu 22.04.1 LTS   5.15.49-linuxkit   containerd://1.6.12
kind-worker          Ready    <none>          165m   v1.26.0   172.18.0.3    
➥ <none>        Ubuntu 22.04.1 LTS   5.15.49-linuxkit   containerd://1.6.12
kind-worker2         Ready    <none>          165m   v1.26.0   172.18.0.2    
➥ <none>        Ubuntu 22.04.1 LTS   5.15.49-linuxkit   containerd://1.6.12
```

我们可以看到 Pod 位于 `kind-worker2` 上，IP 地址为 172.18.0.2。现在让我们使用命令 `curl 172.18.0.2:30000` 来访问应用程序。输出将如下所示：

```
root@kind-control-plane:/# curl 172.18.0.2:30000
Server address: 10.244.2.7:80
Server name: hello-5dc6ddf4c4-qrwq4
Date: 18/Feb/2023:19:47:09 +0000
URI: /
Request ID: fb4b47a4fe62ed734b2c023d802bf46e
```

### D.6.9 安装 Ingress 控制器和 Ingress 资源

根据本练习的说明，使用命令 `k apply -f https://raw.githubusercontent.com/chadmcrowell/acing-the-cka-exam/main/ch_06/nginx-ingress-controller.yaml` 安装 Ingress 控制器。一旦 Ingress 控制器安装完成，我们可以使用命令 `k edit svc hello-svc` 将 `hello-svc` 服务改回 ClusterIP 服务。这将打开服务在 Vim 文本编辑器中，我们可以将类型从 `NodePort` 改回 `ClusterIP`，并确保删除包含节点端口号的行（nodeport: 30000）。最终的 YAML 将如下所示：

```
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hello
  sessionAffinity: None
  type: ClusterIP
```

现在我们可以创建一个 Ingress 资源，这是我在考试期间会利用打开 Kubernetes 文档的地方。从网站[`kubernetes.io/docs`](https://kubernetes.io/docs)打开浏览器，在搜索栏中输入`ingress`并按 Enter 键。点击第一个链接，名为 Ingress，然后在页面右侧点击 Ingress Resource。你可以直接从页面复制粘贴到你的终端。让我们使用命令`vim ingress.yaml`创建一个名为`ingress.yaml`的文件，将 YAML 文件稍微修改一下，并粘贴如下：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hello.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-svc
            port:
              number: 80
```

一旦你设置了 YAML 文件，你可以使用命令`k apply -f ingress.yaml`应用 YAML。然后运行命令`k get ing`列出 hello Ingress 资源。输出将类似于以下内容：

```
root@kind-control-plane:/# k apply -f ingress.yaml
ingress.networking.k8s.io/hello created
root@kind-control-plane:/# k get ing
NAME    CLASS    HOSTS       ADDRESS   PORTS   AGE
hello   <none>   hello.com             80      2s
```

### D.6.10 安装容器网络接口（CNI）

要安装一个不带 CNI 的 kind Kubernetes 集群，请参阅附录 C。一旦你创建了集群并运行了 Docker exec `-it kind-control-plane`命令来访问控制平面节点（该节点已安装`kubectl`），让我们继续这个练习。

要安装一个桥接 CNI，我们首先确保已经安装了 wget。让我们使用命令`apt update; apt install wget`在控制平面和工作节点上安装它。一旦 wget 安装完成，我们将运行命令`wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz`来下载 CNI 插件。我们将使用命令`tar -xvf cni-plugins-linux-amd64-v1.1.1.tgz`解压文件，然后运行命令`mv bridge /opt/cni/bin`将文件移动到 bin 目录。现在我们可以使用命令`kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml`安装 Calico CNI。当你运行命令`k get no`时，你会看到节点的状态从`Not Ready`变为`Ready`。输出将类似于以下内容：

```
root@kind-control-plane:/# k get no
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   11m   v1.25.0-beta.0
kind-worker          Ready    <none>          10m   v1.25.0-beta.0
```

## D.7 第七章考试练习

以下练习基于存储，并且与前面的章节类似，需要按顺序完成。在完成第一个练习之前，你将无法完成第二个练习。对持久卷、持久卷声明和存储类有良好的理解对于考试至关重要。

### D.7.1 创建持久卷

要创建一个持久卷声明（PVC），我们可以参考 Kubernetes 文档，考试期间我们可以将其保持打开状态。这将允许我们直接从页面复制粘贴到我们的终端，这样可以最大限度地利用你在考试中有限的时间。访问 [`kubernetes.io/docs`](https://kubernetes.io/docs) 并使用屏幕左侧的搜索栏搜索术语 *使用持久卷*。点击名为“配置 Pod 使用持久卷进行存储 | Kubernetes”的链接，并滚动到“创建持久卷”部分。该页面的完整 URL 是 [`kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume)。将此页面的 YAML 复制粘贴到一个名为 `pv.yaml` 的新文件中。

我们将持久卷的名称从 `task-pv-volume` 更改为 `volstore308`，并将存储从 `10Gi` 更改为 `22Mi`。保存文件 `pv.yaml` 并使用命令 `k create -f pv.yaml` 创建持久卷。如果你执行 `k get pv` 命令，你将看到类似于以下内容的输出：

```
root@kind-control-plane:/# k get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   
➥ STORAGECLASS
volstore308  100Mi      RWO            Retain           Available           
➥ manual
```

### D.7.2 创建持久卷声明

要创建 PVC，我们将参考之前提到的文档中的同一页，希望你仍然保持它打开。如果没有，这里是有链接：[`kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolumeclaim`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolumeclaim)。将 YAML 复制粘贴到一个名为 `pvc.yaml` 的文件中（使用命令 `vim pvc.yaml`）。

我们将把 PVC 的名称从 `task-pv-claim` 更改为 `pv-claim-vol`，并将存储从 `3Gi` 更改为 `90Mi`。使用命令 `k apply -f pvc.yaml` 创建 PVC。一旦资源创建成功，你可以使用命令 `k get pvc` 查看 PVC，也可以使用命令 `k get pv` 查看持久卷（PV）。命令的输出将如下所示：

```
root@kind-control-plane:/# k get pvc
NAME          STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   
➥ AGE
pv-claim-vol  Bound    vol02833   100Mi      RWO            manual         
➥ 4m
```

### D.7.3 创建使用声明的 Pod

要创建一个名为 `pod-access` 的 Pod 的 YAML，并使用 `centos:7` 镜像，我们将运行命令 `k run pod-access --image centos:7 --dry-run -o yaml > pod-access.yaml`。一旦文件保存，我们可以使用命令 `vim pod-access.yaml` 在 Vim 文本编辑器中打开它。一旦文件打开，我们可以在 `containers:` 下添加将卷通过之前练习中创建的声明附加的章节。在 `volumeMounts:` 下，添加 `- name: vol` 和 `mountPath: /tmp/persistence`。为了保持容器活跃，我们可以在 `- command:` 后面添加 sleep 命令，后面跟着 `- sleep` 和 `- "3600"`。结果将如下所示：

```
  containers:
  - command:
    - sleep
    - "3600"
    image: centos:7
    name: pod-access
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /tmp/persistence
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: pv-claim-vol
```

您可以使用命令 `k apply -f pod-access.yaml` 创建 Pod。

### D.7.4 创建存储类

要创建名为 `node-local` 的存储类，我们可以再次参考 Kubernetes 文档。从 [`kubernetes.io/docs`](https://kubernetes.io/docs)，让我们在搜索栏中搜索存储类。点击页面右侧的“Local”链接的第一个结果，名为 Storage Classes。将页面上的 YAML 直接复制粘贴到您的终端中。我们将完全按照原样粘贴，并将名称更改为 `node-local` 到一个名为 `sc.yaml` 的新文件中。最终结果如下所示：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: node-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

存储类可以使用命令 `k apply -f sc.yaml` 创建。我们可以使用命令 `k get sc` 查看存储类。

### D.7.5 为存储类创建持久卷声明

要为名为 `claim-sc` 的存储类创建 PVC，我们将使用之前创建的 PVC 并对其进行修改。让我们使用命令 `cp pvc.yaml pvc-sc.yaml` 复制该文件。我们将 PVC 的名称更改为 `claim-sc`，将 claim 更改为 `39Mi`，并将访问模式更改为 `ReadWriteOnce`。此 PVC 的最终 YAML 应如下所示：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-sc
spec:
  storageClassName: claim-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 39Mi
```

我们可以使用命令 `k apply -f pvc-sc.yaml` 创建 PVC。

### D.7.6 从存储类创建 Pod

要使用 nginx 镜像创建名为 `pod-sc` 的 Pod，我们可以运行命令 `k run pod-sc --image nginx --dry-run=client -o yaml > pod-sc.yaml`。一旦文件已保存，我们可以使用命令 `vim pod-sc.yaml` 在 Vim 文本编辑器中打开它。一旦文件打开，我们可以在 `containers:` 下添加将通过之前练习中创建的 claim 挂载的卷的部分。在 `volumeMounts:` 下，添加与容器名称和镜像一致的 `volumeMounts:`。在 `volumeMounts:` 下，添加名称 `- name: vol` 和 `mountPath: /tmp/persistence` 在其下方。为了保持容器活跃，我们可以在 `command:` 下方添加 sleep 命令，后跟 `- sleep` 和 `- "3600"`。结果将如下所示：

```
  containers:
    image: nginx
    name: pod-sc
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /tmp/persistence
      command:
        - sleep
        - “3600”
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: claim-sc
```

您可以使用命令 `k apply -f pod-sc.yaml` 创建 Pod。

## D.8 第八章考试练习

第八章全部关于故障排除，因此这些练习中的许多都需要您在集群中模拟某些“损坏”的情况。只需知道，对于考试，这些可能已经以组件已经损坏的方式安排，而在这里我们的实践实验室环境中，我们需要设置场景，这可能需要额外的几个步骤。

### D.8.1 修复 Pod YAML

运行练习中的命令，即 `k run testbox --image busybox --command 'sleep 3600'`。当你这样做时，你会看到状态是 `RunContainerError`。故障排除的第一步是使用命令 `k logs testbox` 检查日志。但结果不会返回任何日志，因此我们必须通过决策树并描述 Pod。为此，我们可以运行命令 `k describe po testbox`。我们可以看到 `describe` 命令的事件，看起来如下所示：

```
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  119s                default-scheduler  Successfully 
➥ assigned default/testbox to kind-worker
  Normal   Pulled     116s                kubelet            Successfully 
➥ pulled image "busybox" in 2.754916379s
  Normal   Pulled     114s                kubelet            Successfully 
➥ pulled image "busybox" in 622.019612ms
  Normal   Pulled     98s                 kubelet            Successfully 
➥ pulled image "busybox" in 710.339798ms
  Normal   Created    73s (x4 over 116s)  kubelet            Created 
➥ container testbox
  Warning  Failed     73s (x4 over 116s)  kubelet            Error: failed 
➥ to create containerd task: failed to create shim task: OCI runtime 
➥ create failed: runc create failed: unable to start container process: 
➥ exec: "sleep 3600": executable file not found in $PATH: unknown
  Normal   Pulled     73s                 kubelet            Successfully 
➥ pulled image "busybox" in 526.759076ms
  Warning  BackOff    45s (x7 over 114s)  kubelet            Back-off 
➥ restarting failed container
  Normal   Pulling    33s (x5 over 118s)  kubelet            Pulling image 
➥ "busybox"
  Normal   Pulled     32s                 kubelet            Successfully 
➥ pulled image "busybox" in 497.474074ms
```

错误是“无法创建 containerd 任务”，这意味着在再次启动容器之前，我们需要修改 YAML。我们可以通过命令 `k edit po testbox` 来编辑 Pod，并更改以 `command` 开头的行。将 `sleep 3600` 替换为下一行上的 `3600`，并用引号和另一个破折号包围。YAML 的最终结果应该是这样的：

```
  containers:
  - command:
    - sleep
    - "3600"
    image: busybox
```

当你尝试保存并退出时，你会得到一个错误消息，但你只需简单地退出并忽略该消息。你会看到文件已经在 `/tmp` 目录中保存了一个副本。你可以用命令 `k replace -f /tmp/kubectl-edit-8848373.yaml` 来替换 Pod YAML。请注意，YAML 文件的名字将和这里的不同。这将创建一个新的 Pod，你现在会看到 Pod 正在运行。

```
root@kind-control-plane:/# k replace -f /tmp/kubectl-edit-4023269864.yaml -
➥ -force
pod "testbox" deleted
pod/testbox replaced
root@kind-control-plane:/# k get po
NAME      READY   STATUS    RESTARTS   AGE
testbox   1/1     Running   0          2s
```

### D.8.2 修复 Pod 镜像

要创建一个名为 `busybox2` 的新容器，使用 `busybox:1.35.0` 镜像，运行命令 `k run busybox2 -image busybox:1.35.0`。通过运行命令 `k get po`，你会看到以下结果：

```
root@kind-control-plane:/# k get po
NAME       READY   STATUS        RESTARTS      AGE
busybox2   0/1     Completed     2 (20s ago)   23s
```

我们可以通过首先尝试获取日志来遵循决策树，但这个容器没有日志。然后我们可以通过命令 `k describe po busybox2` 来描述 Pod。Pod 描述中的事件并没有告诉我们太多，所以它似乎正在正常工作。我们可以通过一个简单的命令来防止容器完成。让我们运行命令 `k run busybox2 --image busybox:1.35.0 -it -sh`，这将打开容器的 shell。当我们退出 shell 时，我们可以运行 `k get po` 并看到容器正在运行。

### D.8.3 修复已完成的 Pod

要创建一个名为 `curlpod2` 的新容器，使用 `nicolaka/netshoot` 镜像，我们将运行命令 `k run curlpod --image nicolaka/netshoot -it --sh`。这将打开容器内的 shell。让我们运行命令 `nslookup kubernetes` 并退出 shell。我们再次看到 Pod 已经完成。这是因为容器镜像中没有命令来保持其运行。我们可以通过命令 `kubectl run curlpod --image=nicolaka/netshoot --command sleep` `--command "3600"` 来添加这个命令。

### D.8.4 修复 Kubernetes 调度器

将文件 `kube-scheduler.yaml` 移动以备份。这总是一个好主意，尤其是在考试时，你不能简单地重建集群。让我们运行命令 `cp /etc/Kubernetes/manifests/kube-scheduler.yaml /tmp/kube-scheduler.yaml`。然后我们可以通过运行命令 `vim /etc/Kubernetes/manifests/kube-scheduler.yaml` 来修改现有的文件。一旦我们在 Vim 文本编辑器中打开它，将 `kube-scheduler` 末尾的额外 `r` 添加到单词末尾，使其变为 `kube-schedulerr`。保存并退出文件。

现在调度器配置已经被修改，让我们通过命令 `k run nginx -image nginx` 来看看是否可以调度一个 Pod，并通过运行命令 `k get po` 来检查 Pod 的状态。输出将如下所示：

```
root@kind-control-plane:/# k run nginx --image nginx
pod/nginx created
root@kind-control-plane:/# k get po
NAME      READY   STATUS    RESTARTS        AGE
curlpod   1/1     Running   1 (6m37s ago)   8m1s
nginx     0/1     Pending   0               1s
```

这是我们知道问题所在，但让我们假装我们不知道的情况之一。使用命令 `k -n kube-system describe po kube-scheduler-kind-control-plane` 查看事件。事件将如下所示：

```
Events:
  Type     Reason   Age                    From     Message
  ----     ------   ----                   ----     -------
  Normal   Created  3m22s (x4 over 4m7s)   kubelet  Created container kube-
➥ scheduler
  Warning  Failed   3m22s (x4 over 4m6s)   kubelet  Error: failed to create 
➥ containerd task: failed to create shim task: OCI runtime create failed: 
➥ runc create failed: unable to start container process: exec: "kube-
➥ schedulerr": executable file not found in $PATH: unknown
  Warning  BackOff  2m53s (x12 over 4m5s)  kubelet  Back-off restarting 
➥ failed container
  Normal   Pulled   2m41s (x5 over 4m7s)   kubelet  Container image 
➥ "registry.k8s.io/kube-scheduler:v1.25.0-beta.0" already present on machine
```

你还会注意到 Pod 处于 `crashloopbackoff` 状态。你可以看到错误信息“无法启动容器进程。”

### D.8.5 修复 kubelet

让我们运行命令 `curl https://raw.githubusercontent.com/chadmcrowell/acing-the-cka-exam/main/ch_08/10-kubeadm.conf --silent --output /etc/systemd/system/kubelet.service.d/10-kubeadm.conf; systemctl daemon-reload; systemctl restart kubelet.` 这将以一种影响 kubelet 的方式破坏集群。让我们使用命令 `systemctl status kubelet` 检查 kubelet 服务的状态。我们注意到服务处于非活动状态，并看到进程存在错误。让我们检查 `/etc/systemd/system/kubelet.service.d` 目录下的文件，该文件名为 `10-kubeadm.conf`。我们会注意到文件中有一个不正确的 `bin` 目录，它应该是 `usr/bin` 而不是 `/usr/local/bin`。让我们将文件更改为 `/usr/bin` 并看看是否可以修复问题。使用命令 `vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf` 打开文件。一旦文件打开，我们可以按照以下方式更改内容：

```
# This eventually leads to kubelet failing to start, see: 
➥ https://github.com/kubernetes-sigs/kind/issues/2323
ExecStartPre=/bin/sh -euc "if [ ! -f /sys/fs/cgroup/cgroup.controllers ] && 
➥ [ ! -d /sys/fs/cgroup/systemd/kubelet ]; then mkdir -p 
➥ /sys/fs/cgroup/systemd/kubelet; fi"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS 
➥ $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cgroup-root=/kubelet
```

当你的文件看起来像这样时，保存并退出文件。然后使用命令 `systemctl daemon-reload` 重新加载守护进程，随后使用命令 `systemctl restart kubelet` 重新启动 kubelet 服务。使用命令 `systemctl status kubelet` 检查 kubelet 服务的状态，现在服务应该处于活动状态并正在运行。
