# 第二章：创建 Kubernetes 集群

在本章中，我们讨论了多种设置完整 Kubernetes 集群的方法。我们涵盖了低级别的标准化工具（`kubeadm`），这些工具也是其他安装程序的基础，并向您展示了控制平面和工作节点的相关二进制文件位置。我们演示了如何编写 systemd 单元文件以监控 Kubernetes 组件，并最终展示了如何在 Google Cloud 平台和 Azure 上设置集群。

# 2.1 为 Kubernetes 集群准备新节点

## 问题

您希望为创建新的 Kubernetes 集群或将其添加到现有集群的节点准备一个新的节点，并配置所有必需的工具。

## 解决方案

要为基于 Ubuntu 的主机准备 Kubernetes 集群，首先需要启用 IPv4 转发并启用 iptables 以查看桥接流量：

```
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

$ sudo sysctl --system
```

为了与`kubeadm`工具兼容，节点上需要关闭交换分区：

```
$ sudo apt install cron -y
$ sudo swapoff -a
$ (sudo crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | sudo crontab -
|| true
```

集群节点需要实现 Kubernetes 容器运行时接口（CRI）。[cri-o](https://cri-o.io) 就是其中之一。cri-o 的版本应与 Kubernetes 版本匹配。例如，如果您要引导一个 Kubernetes 1.27 集群，请相应地配置 `VERSION` 变量：

```
$ VERSION="1.27"
$ OS="xUbuntu_22.04"

$ cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:
stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:
/stable/$OS/ /
EOF

$ cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:
stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:
/stable:/cri-o:/$VERSION/$OS/ /
EOF

$ curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:
/stable:/cri-o:/$VERSION/$OS/Release.key | \
    sudo apt-key add -
$ curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:
/stable/$OS/Release.key | \
    sudo apt-key add -

$ sudo apt-get update

$ sudo apt-get install cri-o cri-o-runc cri-tools -y
```

然后重新加载 systemd 配置并启用 cri-o：

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable crio --now
```

`kubeadm` 工具用于从头开始引导 Kubernetes 集群，以及加入现有集群。使用以下命令启用其软件仓库：

```
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/k8s-archive-keyring.gpg]
https://apt.kubernetes.io/
kubernetes-xenial main
EOF

$ sudo apt-get install -y apt-transport-https ca-certificates curl
$ sudo curl -fsSLo /etc/apt/keyrings/k8s-archive-keyring.gpg \
    https://dl.k8s.io/apt/doc/apt-key.gpg

$ sudo apt-get update
```

现在您可以安装所有引导 Kubernetes 集群节点所需的工具。您将需要以下工具：

+   `kubelet` 二进制文件

+   `kubeadm` 命令行界面

+   `kubectl` 客户端

运行此命令以安装它们：

```
$ sudo apt-get install -y kubelet kubeadm kubectl
```

然后将这些软件包标记为被拒绝更新，以防止它们被自动升级：

```
$ sudo apt-mark hold kubelet kubeadm kubectl
```

您的 Ubuntu 主机现在已准备好成为 Kubernetes 集群的一部分。

## 讨论

`kubeadm` 是一个设置工具，提供了 `kubeadm init` 和 `kubeadm join`。`kubeadm init` 用于引导 Kubernetes 控制平面节点，而 `kubeadm join` 用于引导工作节点并将其加入集群。实质上，`kubeadm` 提供了使得最小可行集群运行起来所需的动作。`kubelet` 是在每个节点上运行的 *节点代理*。

除了 cri-o 外，还有其他容器运行时值得研究的是 [containerd](https://oreil.ly/M1kDx)、[Docker Engine](https://oreil.ly/P5_l_) 和 [Mirantis Container Runtime](https://oreil.ly/BEWaG)。

# 2.2 引导 Kubernetes 控制平面节点

## 问题

您已经为 Kubernetes 初始化了一个 Ubuntu 主机（见配方 2.1），现在需要引导一个新的 Kubernetes 控制平面节点。

## 解决方案

安装了`kubeadm`二进制文件后，您已准备好开始引导 Kubernetes 集群。在节点上初始化控制平面，使用以下命令：

```
$ NODENAME=$(hostname -s)
$ IPADDR=$(ip route get 8.8.8.8 | sed -n 's/.*src \([^\ ]*\).*/\1/p')
$ POD_CIDR=192.168.0.0/16
```

###### 警告

控制平面节点应至少具有两个虚拟 CPU 和 2 GB 内存。

现在使用 `kubeadm` 初始化控制平面节点：

```
$ sudo kubeadm init --apiserver-advertise-address=$IPADDR \
    --apiserver-cert-extra-sans=$IPADDR  \
    --pod-network-cidr=$POD_CIDR \
    --node-name $NODENAME \
    --ignore-preflight-errors Swap
[init] Using Kubernetes version: v1.27.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
...
```

`init`命令的输出包含配置信息，用于设置`kubectl`与您的集群通信。一旦配置了`kubectl`，您可以使用以下命令验证集群组件的健康状态：

```
$ kubectl get --raw='/readyz?verbose'
```

要获取集群信息，请使用：

```
$ kubectl cluster-info
```

## 讨论

用户工作负载不会安排在控制平面节点上执行。如果您正在创建一个实验性的单节点集群，那么您需要对控制平面节点进行标记，以便在控制平面节点上安排用户工作负载：

```
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 参见

+   [使用 `kubeadm` 创建集群](https://oreil.ly/q9nwI)

# 2.3 安装用于集群网络的容器网络插件

## 问题

您已经引导了一个 Kubernetes 控制平面节点（参见配方 2.2），现在需要安装一个 pod 网络插件，以便 pod 可以彼此通信。

## 解决方案

您可以在控制平面节点上使用以下命令安装 Calico 网络插件：

```
$ kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/
v3.26.1/manifests/calico.yaml
```

## 讨论

您必须使用与您的集群兼容并且符合您需求的容器网络接口（CNI）插件。有许多插件实现了 CNI。查看 Kubernetes 文档中关于[可用插件的非详尽列表](https://oreil.ly/HosU6)。

# 2.4 向 Kubernetes 集群添加工作节点

## 问题

您已经初始化了 Kubernetes 控制平面节点（参见配方 2.2），并安装了一个 CNI 插件（参见配方 2.3），现在您想要向集群添加工作节点。

## 解决方案

使用 Ubuntu 主机初始化 Kubernetes，如配方 2.1 所示，在控制平面节点上执行以下命令以显示集群`join`命令：

```
$ kubeadm token create --print-join-command
```

现在，在工作节点上执行`join`命令：

```
$ sudo kubeadm join --token *<token>*
```

###### 警告

工作节点应至少具有一个 vCPU 和 2 GB RAM。

返回到您的控制平面节点终端会话，您将看到您的节点加入：

```
$ kubectl get nodes
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   28m   v1.27.2
worker   Ready    <none>          10s   v1.27.2

```

您可以重复这些步骤以向 Kubernetes 集群添加更多的工作节点。

## 讨论

工作节点是您的工作负载运行的地方。当集群开始耗尽资源时，您会开始注意到新 pod 的*挂起*状态。此时，您应该考虑通过添加更多的工作节点来增加集群资源。

# 2.5 部署 Kubernetes 仪表板

## 问题

您已经创建了一个 Kubernetes 集群，现在您想要使用用户界面在集群上创建、查看和管理容器化工作负载。

## 解决方案

使用 [Kubernetes 仪表板](https://oreil.ly/n7WQw)，这是一个基于 Web 的用户界面，用于将容器化应用程序部署到 Kubernetes 集群，并管理集群资源。

###### 提示

如果您使用 Minikube，可以通过启用`dashboard`插件简单地安装 Kubernetes 仪表板：

```
$ minikube addons enable dashboard

```

要部署 v2.7.0 Kubernetes 仪表板，请执行以下操作：

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/
v2.7.0/aio/deploy/recommended.yaml

```

然后验证部署是否准备就绪：

```
$ kubectl get deployment kubernetes-dashboard -n kubernetes-dashboard
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1/1     1            1           44s

```

# 2.6 访问 Kubernetes 仪表板

## 问题

您已在集群上安装了 Kubernetes 仪表板（参见 Recipe 2.5），并希望从 Web 浏览器访问仪表板。

## 解决方案

您需要创建一个[`ServiceAccount`](https://oreil.ly/pXErB)，具有管理集群权限。创建名为*sa.yaml*的文件，并包含以下内容：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

使用此命令创建`ServiceAccount`：

```
$ kubectl apply -f sa.yaml

```

要访问 Kubernetes 仪表板，您需要创建与此帐户关联的认证令牌。保存在以下命令的输出中打印的令牌：

```
$ kubectl -n kubernetes-dashboard create token admin-user
eyJhbGciOiJSUzI1NiIsImtpZCI6...

```

由于 Kubernetes 仪表板是集群本地服务，您需要设置与集群的代理连接：

```
$ kubectl proxy

```

访问[*http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/workloads?namespace=_all*](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/workloads?namespace=_all)，您现在可以打开 Kubernetes 仪表板并使用之前创建的认证令牌进行身份验证。

在浏览器中打开的 UI 中，您将看到图 2-1 中所示的页面。

![仪表板应用程序创建视图的快照](img/kcb2_0201.png)

###### 图 2-1\. 仪表板应用程序创建视图的快照

###### 提示

如果您使用 Minikube，您只需执行以下操作：

```
$ minikube dashboard

```

## 讨论

要创建一个应用程序，请点击右上角的加号（+），选择“从表单创建”选项卡，给应用程序命名，并指定您要使用的容器镜像。然后点击部署按钮，您将看到一个新视图，显示部署、pod 和副本集。在 Kubernetes 中有数十种重要的资源类型，例如部署、pod、副本集、服务等，我们将在本书的其余部分详细探讨。

在创建 Redis 容器后，图 2-2 中的快照展示了典型的仪表板视图。

![带有 Redis 应用程序的仪表板概览](img/kcb2_0202.png)

###### 图 2-2\. 带有 Redis 应用程序的仪表板概览

如果您返回到终端会话并使用命令行客户端，则会看到相同的内容：

```
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/redis-584fd7c758-vwl52   1/1     Running   0          5m9s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   19m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/redis   1/1     1            1           5m9s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/redis-584fd7c758   1         1         1       5m9s

```

您的 Redis pod 将运行 Redis 服务器，如下面的日志所示：

```
$ kubectl logs redis-3215927958-4x88v
...
1:C 25 Aug 2023 06:17:23.934 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 25 Aug 2023 06:17:23.934 * Redis version=7.2.0, bits=64, commit=00000000,
modified=0, pid=1, just started
1:C 25 Aug 2023 06:17:23.934 # Warning: no config file specified, using the
default config. In order to specify a config file use redis-server
/path/to/redis.conf
1:M 25 Aug 2023 06:17:23.934 * monotonic clock: POSIX clock_gettime
1:M 25 Aug 2023 06:17:23.934 * Running mode=standalone, port=6379.
1:M 25 Aug 2023 06:17:23.935 * Server initialized
1:M 25 Aug 2023 06:17:23.935 * Ready to accept connections tcp
```

# 2.7 部署 Kubernetes Metrics Server

## 问题

您已经部署了 Kubernetes 仪表板（参见 Recipe 2.5），但在仪表板中看不到 CPU 和内存使用信息。

## 解决方案

Kubernetes 仪表板需要[Kubernetes Metrics Server](https://oreil.ly/BEHwR)来可视化 CPU 和内存使用情况。

###### 提示

如果您使用 Minikube，只需启用`metrics-server`附加组件即可安装 Kubernetes Metrics Server：

```
$ minikube addons enable metrics-server

```

要部署最新版本的 Kubernetes Metrics Server，请执行以下操作：

```
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/
latest/download/components.yaml

```

然后验证部署是否就绪：

```
$ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           7m27s

```

如果看到部署未进入就绪状态，请检查 pod 日志：

```
$ kubectl logs -f deployment/metrics-server -n kube-system
I0707 05:06:19.537981   1 server.go:187] "Failed probe"
probe="metric-storage-ready" err="no metrics to serve"
E0707 05:06:26.395852   1 scraper.go:140] "Failed to scrape node" err="Get
\"https://192.168.64.50:10250/metrics/resource\": x509: cannot validate
certificate for 192.168.64.50 because it doesn't contain any IP SANs"
node="minikube"

```

如果你看到错误消息“无法验证证书”，则需要将标志 `--kubelet-insecure-tls` 添加到 Metrics Server 部署中：

```
$ kubectl patch deployment metrics-server -n kube-system --type='json'
-p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":
"--kubelet-insecure-tls"}]'

```

###### 提示

Metrics Server 启动后，可能需要几分钟才能使其可用。如果还没有准备好状态，则请求指标可能会产生错误。

一旦 Metrics Server 启动，Kubernetes 仪表板将显示 CPU 和内存使用统计信息，如 Figure 2-3 所示。

![仪表板节点视图快照](img/kcb2_0203.png)

###### Figure 2-3\. 仪表板集群节点视图

## 讨论

使用 `kubectl top` 命令也可以在命令行中查看节点和 Pod 的指标：

```
$ kubectl top pods -A
NAMESPACE     NAME                               CPU(cores)   MEMORY(bytes)
kube-system   coredns-5d78c9869d-5fh78           9m           9Mi
kube-system   etcd-minikube                      164m         36Mi
kube-system   kube-apiserver-minikube            322m         254Mi
kube-system   kube-controller-manager-minikube   123m         35Mi
kube-system   kube-proxy-rvl8v                   13m          12Mi
kube-system   kube-scheduler-minikube            62m          15Mi
kube-system   storage-provisioner                22m          7Mi

```

类似地，要查看节点指标，请执行以下操作：

```
$ kubectl top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   415m         10%    1457Mi          18%

```

## 参见

+   [Kubernetes Metrics Server GitHub 仓库](https://oreil.ly/C_O6W)

+   [资源指标流水线文档](https://oreil.ly/ODZCr)

# 2.8 从 GitHub 下载 Kubernetes 版本

## 问题

你想要下载官方发布的 Kubernetes 版本，而不是从源代码编译。

## 解决方案

Kubernetes 项目为每个版本发布一个归档。你可以在特定版本的 CHANGELOG 文件中找到归档的链接。转到项目页面的 *CHANGELOG* 文件夹，并打开你选择的版本的 CHANGELOG 文件。在文件中，你会找到一个链接，用于下载该版本的 *kubernetes.tar.gz* 文件。

例如，如果你想要下载 v1.28.0 版本，可以打开 *CHANGELOG-1.28.md*，在标题为“Downloads for v1.28.0”的部分中，你将找到下载 *kubernetes.tar.gz* 的链接（[*https://dl.k8s.io/v1.28.0/kubernetes.tar.gz*](https://dl.k8s.io/v1.28.0/kubernetes.tar.gz)）。

```
$ wget https://dl.k8s.io/v1.28.0/kubernetes.tar.gz
```

如果你想从源代码编译 Kubernetes，请参见 Recipe 15.1。

## 讨论

CHANGELOG 文件还列出了 *kubernetes.tar.gz* 归档的 `sha512 哈希值`。建议你验证本地下载的 *kubernetes.tar.gz* 归档的完整性，以确保其未被任何方式篡改。为此，请生成本地下载归档的 `sha512` 哈希值，并与 CHANGELOG 中列出的哈希值进行比较：

```
$ sha512sum kubernetes.tar.gz
9aaf7cc004d09297dc7bbc1f0149....  kubernetes.tar.gz

```

# 2.9 下载客户端和服务器二进制文件

## 问题

你已经下载了一个发布归档（参见 Recipe 2.8），但它不包含实际的二进制文件。

## 解决方案

发布归档不包含发布二进制文件（为了保持发布归档的小巧性）。因此，你需要单独下载二进制文件。为此，请运行 *get-kube-binaries.sh* 脚本，如下所示：

```
$ tar -xvf kubernetes.tar.gz
$ cd kubernetes/cluster
$ ./get-kube-binaries.sh
```

完成后，你将在 *client/bin* 中获得客户端二进制文件：

```
$ ls ../client/bin
kubectl		kubectl-convert
```

并且包含服务器二进制文件的归档位于 *server/kubernetes* 文件夹中：

```
$ ls ../server/kubernetes
kubernetes-server-linux-amd64.tar.gz   kubernetes-manifests.tar.gz	  README
...
```

## 讨论

如果您想跳过下载整个发布存档并快速下载客户端和服务器二进制文件，您可以直接从[下载 Kubernetes](https://oreil.ly/tdN0P)获取它们。在此页面上，您将找到各种操作系统和架构组合的二进制文件的直接链接，如 Figure 2-4 所示。

![下载 kubernetes.com 的屏幕截图，列出了适用于 Darwin 操作系统的 k8s v1.28.0 版本的二进制文件](img/kcb2_0204.png)

###### 图 2-4\. downloadkubernetes.com，列出了适用于 Darwin 操作系统的 Kubernetes v1.28.0 版本的二进制文件

# 2.10 使用 systemd 单元文件运行 Kubernetes 组件

## 问题

你已经使用了 Minikube（参见 Recipe 1.2）进行学习，并且知道如何使用`kubeadm`引导一个 Kubernetes 集群（参见 Recipe 2.2），但你想要从头开始安装一个集群。

## 解决方案

为此，您需要使用 systemd 单元文件运行 Kubernetes 组件。您只寻找通过 systemd 运行`kubelet`的基本示例。

检查`kubeadm`如何通过 systemd 单元文件配置 Kubernetes 守护进程的启动方式，可以帮助你理解如何自行操作。如果你仔细查看`kubeadm`的配置，你会发现`kubelet`在集群中的每个节点上（包括控制平面节点）都在运行。

这是一个示例，您可以通过登录到任何使用`kubeadm`构建的集群节点（参见 Recipe 2.2）来复制：

```
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled;
     vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Tue 2023-05-30 04:21:29 UTC; 2h 49min ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 797 (kubelet)
      Tasks: 11 (limit: 2234)
     Memory: 40.2M
        CPU: 5min 14.792s
     CGroup: /system.slice/kubelet.service
             └─797 /usr/bin/kubelet \
                --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
                --kubeconfig=/etc/kubernetes/kubelet.conf \
                --config=/var/lib/kubelet/config.yaml \
                --container-runtime-endpoint=unix:///var/run/crio/crio.sock \
                --pod-infra-container-image=registry.k8s.io/pause:3.9
```

这为你提供了 systemd 单元文件在 */lib/systemd/system/kubelet.service* 中的链接以及在 */etc/systemd/system/kubelet.service.d/10-kubeadm.conf* 中的配置。

单元文件很简单——它指向了 */usr/bin* 中安装的 `kubelet` 二进制文件：

```
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/home/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

配置文件告诉您如何启动`kubelet`二进制文件：

```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/
bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/default/kubelet

ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS
$KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

所有指定的选项，如`--kubeconfig`，由环境变量`$KUBELET_CONFIG_ARGS`定义，都是`kubelet`二进制文件的启动选项。

## 讨论

[systemd](https://oreil.ly/RmuZp) 是一个系统和服务管理器，有时被称为*初始化系统*。现在已成为 Ubuntu 和 CentOS 上的默认服务管理器。

刚才展示的单元文件仅处理`kubelet`。您可以为 Kubernetes 集群的所有其他组件（即 API 服务器、控制器管理器、调度器、代理）编写自己的单元文件。[Kubernetes the Hard Way](https://oreil.ly/AWnxD) 提供了每个组件的单元文件示例。

然而，你只需要运行`kubelet`。确实，注意到配置选项 `--pod-manifest-path` 允许你传递一个目录，kubelet 将在其中查找并自动启动 manifests。使用 `kubeadm`，这个目录用于传递 API 服务器、调度器、etcd 和控制器管理器的 manifests。因此，Kubernetes 自行管理，并且 systemd 仅管理`kubelet`进程。

为了说明这一点，您可以列出基于 `kubeadm` 的集群中 */etc/kubernetes/manifests* 目录的内容：

```
$ ls -l /etc/kubernetes/manifests
total 16
-rw------- 1 root root 2393 May 29 11:04 etcd.yaml
-rw------- 1 root root 3882 May 29 11:04 kube-apiserver.yaml
-rw------- 1 root root 3394 May 29 11:04 kube-controller-manager.yaml
-rw------- 1 root root 1463 May 29 11:04 kube-scheduler.yaml
```

查看 *etcd.yaml* 清单的详细信息，您可以看到它是一个运行 etcd 的单容器 `Pod`：

```
$ cat /etc/kubernetes/manifests/etcd.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://10.10.100.30:2379
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.10.100.30:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://10.10.100.30:2380
    - --initial-cluster=master=https://10.10.100.30:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.10.100.30:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://10.10.100.30:2380
    - --name=master
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: registry.k8s.io/etcd:3.5.7-0
    ...

```

## 参见

+   [`kubelet` 配置选项](https://oreil.ly/E95yp)

# 2.11 在 Google Kubernetes Engine 上创建 Kubernetes 集群

## 问题

您想在 Google Kubernetes Engine (GKE) 上创建一个 Kubernetes 集群。

## 解决方案

要使用 GKE，您首先需要几件事：

+   一个启用了结算的[Google 云平台 (GCP)](https://oreil.ly/CAiDf) 账户

+   一个启用了[GKE](https://oreil.ly/eGX2n) 的 GCP 项目

+   安装了[Google Cloud SDK](https://oreil.ly/Y00rC)

Google Cloud SDK 包含用于从命令行与 GCP 服务交互的 `gcloud` CLI 工具。安装 SDK 后，使用身份验证 `gcloud` 访问您的 GCP 项目：

```
$ gcloud auth login
```

使用 `gcloud` 命令行界面，使用 `container clusters create` 命令创建 Kubernetes 集群，如下所示：

```
$ gcloud container clusters create oreilly --zone us-east1-b
```

默认情况下，这将在指定的区域或区域中创建一个具有三个工作节点的 Kubernetes 集群。主节点由 GKE 服务管理，无法访问。

###### 提示

如果不确定用于 `--zone` 或 `--region` 参数的[区域或地区](https://oreil.ly/4Bvua)，执行 `gcloud compute zones list` 或 `gcloud compute regions list` 并选择附近的一个。通常，区域比地区需要的资源更少。

使用完集群后，请不要忘记将其删除以避免收费：

```
$ gcloud container clusters delete oreilly --zone us-east1-b
```

## 讨论

您可以通过使用[Google Cloud Shell](https://oreil.ly/E-Qcr)，一个纯在线基于浏览器的解决方案，跳过 `gcloud` CLI 的安装。

您可以使用此命令列出您现有的 GKE 集群：

```
$ gcloud container clusters list --zone us-east1-b
NAME     ZONE        MASTER_VERSION     MASTER_IP     ...  STATUS
oreilly  us-east1-b  1.24.9-gke.2000    35.187.80.94  ...  RUNNING
```

###### 注意

`gcloud` CLI 允许您调整、更新和升级您的集群：

```
...
COMMANDS
...
     resize
        Resizes an existing cluster for running
        containers.
     update
        Update cluster settings for an existing container
        cluster.
     upgrade
        Upgrade the Kubernetes version of an existing
        container cluster.
```

## 参见

+   GKE [快速入门](https://oreil.ly/WMDSx)

+   Google Cloud Shell [快速入门](https://oreil.ly/_w0va)

# 2.12 在 Azure Kubernetes 服务上创建 Kubernetes 集群

## 问题

您想在 Azure Kubernetes Service (AKS) 上创建一个 Kubernetes 集群。

## 解决方案

要创建 AKS 集群，您需要以下内容：

+   一个[Microsoft Azure 门户账户](https://oreil.ly/PyUA0)

+   安装了[Azure CLI](https://oreil.ly/An7xM)

首先确保安装了 Azure CLI 版本 2.0 或更高，并登录到 Azure：

```
$ az --version | grep "^azure-cli"
azure-cli                         2.50.0 *

$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and
enter the code XXXXXXXXX to authenticate.
[
  {
    "cloudName": "AzureCloud",
    "id": "****************************",
    "isDefault": true,
    "name": "Free Trial",
    "state": "Enabled",
    "tenantId": "*****************************",
    "user": {
      "name": "******@hotmail.com",
      "type": "user"
    }
  }
]
```

创建名为 `k8s` 的 Azure 资源组，以容纳所有您的 AKS 资源，如 VM 和网络组件，并且便于稍后进行清理和拆除：

```
$ az group create --name k8s --location northeurope
{
  "id": "/subscriptions/************************/resourceGroups/k8s",
  "location": "northeurope",
  "managedBy": null,
  "name": "k8s",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

###### 提示

如果不确定用于 `--location` 参数的[区域](https://oreil.ly/fdGdc)，执行 `az account list-locations` 并选择附近的一个。

现在，您已设置了资源组 `k8s`，可以像这样创建具有一个工作节点（在 Azure 术语中称为 *agent*）的集群：

```
$ az aks create -g k8s -n myAKSCluster --node-count 1 --generate-ssh-keys
{
  "aadProfile": null,
  "addonProfiles": null,
  "agentPoolProfiles": 
    {
      "availabilityZones": null,
      "count": 1,
      "creationData": null,
      "currentOrchestratorVersion": "1.26.6",

```

请注意，`az aks create` 命令可能需要几分钟才能完成。完成后，该命令将返回一个包含有关已创建集群信息的 JSON 对象。

因此，在 Azure 门户中，您应该看到类似于 [Figure 2-5 的东西。首先找到 `k8s` 资源组，然后导航到部署选项卡。

![Azure 门户的屏幕截图，显示位于 `k8s` 资源组中的 AKS 集群](img/kcb2_0205.png)

###### 图 2-5\. Azure 门户，显示位于 `k8s` 资源组中的 AKS 集群

您现在可以连接到集群：

```
$ az aks get-credentials --resource-group k8s --name myAKSCluster
```

您现在可以在环境中四处查看并验证设置：

```
$ kubectl cluster-info
Kubernetes master is running at https://k8scb-k8s-143f1emgmt.northeurope.cloudapp
  .azure.com
Heapster is running at https://k8scb-k8s-143f1emgmt.northeurope.cloudapp.azure
  .com/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://k8scb-k8s-143f1emgmt.northeurope.cloudapp.azure
  .com/api/v1/namespaces/kube-system/services/kube-dns/proxy
kubernetes-dashboard is running at https://k8scb-k8s-143f1emgmt.northeurope
  .cloudapp.azure.com/api/v1/namespaces/kube-system/services/kubernetes-dashboard
  /proxy
tiller-deploy is running at https://k8scb-k8s-143f1emgmt.northeurope.cloudapp
  .azure.com/api/v1/namespaces/kube-system/services/tiller-deploy/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-78916010-vmss000000   Ready    agent   26m   v1.24.9
```

事实上，正如您从输出中所见，我们已经创建了一个单节点集群。

###### 提示

如果您不想或无法安装 Azure CLI，则可以使用来自浏览器的 [Azure Cloud Shell](https://oreil.ly/IUFJQ) 作为替代方案。

当你完成发现 AKS 后，请不要忘记通过删除资源组 `k8s` 来关闭集群并删除所有资源：

```
$ az group delete --name k8s --yes --no-wait
```

尽管 `az group delete` 命令立即返回，由于存在 `--no-wait` 标志，但可能需要长达 10 分钟才能删除所有资源并实际销毁资源组。您可能需要在 Azure 门户中检查以确保一切按计划进行。

## 参见

+   在 Microsoft Azure 文档中的 [“快速入门：使用 Azure CLI 部署 Azure Kubernetes Service 集群”](https://oreil.ly/YXv3B)

# 2.13 在 Amazon Elastic Kubernetes Service 上创建 Kubernetes 集群

## 问题

您想要在 Amazon Elastic Kubernetes Service (EKS) 上创建一个 Kubernetes 集群。

## 解决方案

要在 Amazon EKS 中创建集群，您需要以下内容：

+   一个 [Amazon Web Services](https://aws.amazon.com) 帐户

+   [AWS CLI](https://aws.amazon.com/cli) 已安装

+   [eksctl](https://eksctl.io) 已安装的 CLI 工具

安装 AWS CLI 后，请[验证客户端身份](https://oreil.ly/_6VMv)以访问您的 AWS 帐户：

```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: eu-central-1
Default output format [None]:

```

eksctl 工具是 Amazon EKS 的官方 CLI。它使用您配置的 AWS 凭据来验证 AWS。使用 eksctl，创建集群：

```
$ eksctl create cluster --name oreilly --region eu-central-1
2023-08-29 13:21:12 [i]  eksctl version 0.153.0-dev+a79b3826a.2023-08-18T...
2023-08-29 13:21:12 [i]  using region eu-central-1
...
2023-08-29 13:36:52 [✔]  EKS cluster "oreilly" in "eu-central-1" region is ready

```

默认情况下，eksctl 在指定区域创建一个具有两个工作节点的集群。您可以通过指定 `--nodes` 标志来调整此参数。

###### 提示

为了获得最低的延迟，请选择距离您最近的 [AWS 区域](https://oreil.ly/Kc9GZ)。

当您不再需要 EKS 集群时，请将其删除以避免为未使用的资源收费：

```
$ eksctl delete cluster oreilly --region eu-central-1
```

## 参见

+   eksctl [介绍](https://eksctl.io/getting-started)

+   [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks)
