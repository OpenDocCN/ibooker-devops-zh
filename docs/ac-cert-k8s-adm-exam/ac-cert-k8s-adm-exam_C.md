# 附录 C. 在 kind 集群中安装 CNI

本附录展示了如何在你的 kind 集群中安装一个新的 CNI。我们将安装 Flannel 和 Calico，这两个都用于 CKA 考试，我们将逐步说明如何在 kind 集群内完成这个过程。这包括创建一个没有 CNI 的 kind 集群，安装网桥 CNI 插件，然后安装 Flannel 或 Calico。

## C.1 不使用 CNI 创建 kind 集群

在我们创建一个新的 Kubernetes 集群之前，我们必须首先创建一个 YAML 文件，我们可以使用这个文件作为 `kind create` 命令的输入。创建一个名为 `config.yaml` 的文件，并将内容粘贴如下：

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
role: control-plane
role: worker
```

现在，运行命令 `kind create cluster --image kindest/node:v1.25.0-beta.0 --config config.yaml` 来根据我们在 `config.yaml` 文件中指定的配置创建一个 kind 集群。输出将类似于以下内容：

```
$ kind create cluster --image kindest/node:v1.25.0-beta.0 --config 
➥ config.yaml
 [21:44:59]
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.0-beta.0) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! 
➥ https://kind.sigs.k8s.io/#community 🙂
```

## C.2 安装网桥 CNI 插件

现在我们已经创建了集群，让我们通过命令 `docker exec -it kind-control-plane bash` 和 `docker exec -it kind-worker bash` 分别获取到两个节点容器的 shell，一次一个。一旦你有了 Bash shell，就可以在 `kind-control-plane` 和 `kind-worker` 上运行命令 `apt update; apt install wget`。这将安装 wget，这是一个命令行工具，我们可以用它从网络上下载文件，这就是我们将使用 `wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz` 命令所做的事情。因为这是一个 tarball 文件，你需要使用命令 `tar -xvf cni-plugins-linux-amd64-v1.1.1.tgz` 来解压它。输出将类似于以下内容：

```
root@kind-control-plane:/# tar -xvf cni-plugins-linux-amd64-v1.1.1.tgz
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

网桥文件对于我们来说是最重要的，因为它将为 Kubernetes 提供使用 Flannel 作为 CNI 所必需的插件。此文件还需要位于特定的目录中，以便被集群识别。那个目录是 `/opt/cni/bin/`，因此我们将使用命令 `mv bridge /opt/cni/bin/` 将文件 bridge 移动到那个目录。

## C.3 安装 Flannel CNI

现在我们已经在 `kind-control-plane` 和 `kind-worker` 上安装了网桥 CNI 插件，我们可以在控制平面节点的 shell 内创建 Flannel Kubernetes 对象来安装 Flannel CNI，命令为 `kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml`。你可以使用命令 `kubectl get no` 验证节点现在是否处于就绪状态。输出将类似于以下内容：

```
root@kind-control-plane:/# kubectl get no
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   16m   v1.25.0-beta.0
kind-worker          Ready    <none>          16m   v1.25.0-beta.0
```

我们也可以使用命令 `kubectl get po -A` 验证 CoreDNS Pods 是否正在运行，以及 Flannel Pods 是否已创建并正在运行。输出将类似于以下内容：

```
root@kind-control-plane:/# k get po -A
NAMESPACE            NAME                                         READY   
➥ STATUS 
kube-flannel         kube-flannel-ds-d6v6t                        1/1     
➥ Running
kube-flannel         kube-flannel-ds-h7b5v                        1/1     
➥ Running
kube-system          coredns-565d847f94-txdvw                     1/1     
➥ Running
kube-system          coredns-565d847f94-vb4kg                     1/1     
➥ Running
kube-system          etcd-kind-control-plane                      1/1     
➥ Running
kube-system          kube-apiserver-kind-control-plane            1/1     
➥ Running
kube-system          kube-controller-manager-kind-control-plane   1/1     
➥ Running
kube-system          kube-proxy-9hsvk                             1/1     
➥ Running
kube-system          kube-proxy-gkvrz                             1/1     
➥ Running
kube-system          kube-scheduler-kind-control-plane            1/1     
➥ Running
local-path-storage   local-path-provisioner-684f458cdd-8bwkh      1/1     
➥ Running
```

这将完成 Flannel 安装的设置。

## C.4 创建一个新的集群类型

安装 Calico CNI 与安装 Flannel 非常相似，只是用于创建 Kubernetes 对象的 YAML 文件不同。因此，再次浏览 C.1 节和 C.2 节，然后从那里继续。如果你已经有一个 kind 集群在运行，你可以执行命令`kind delete cluster`来删除现有的集群，或者你可以使用命令`kind create cluster --image kindest/node:v1.25.0-beta.0 --config config.yaml --name cka`在现有集群旁边创建一个名为`cka`的新集群。你将看到类似于以下输出的内容：

```
$ kind create cluster --image kindest/node:v1.25.0-beta.0 --config 
➥ config.yaml 
--name cka
➥ [9:39:37]
Creating cluster "cka" ...
 ✓ Ensuring node image (kindest/node:v1.25.0-beta.0) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-cka"
You can now use your cluster with:

kubectl cluster-info --context kind-cka

Thanks for using kind! 😊
```

如果你选择在已有的集群旁边安装一个新的集群以获取到节点 shell，你必须使用正确的前缀来引用它们（例如，`cka`）。例如，如果你在跟随教程并选择了`cka`作为你的集群名称，要获取控制平面节点的 shell，你应该输入`docker exec -it cka-control-plane bash`。要获取工作节点的 shell，你应该输入`docker exec -it cka-worker bash`。现在我们已经跟上了进度，让我们继续安装 Calico 作为我们的 CNI。

## C.5 安装 Calico CNI

从 C.2 节开始，那里你安装了 Calico CNI 插件，现在让我们安装实现 Calico CNI 所需的 Kubernetes 对象。在你控制平面的 shell 中（例如，`docker exec -it cka-control-plane bash`），你可以使用命令`kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml`来创建这些 Kubernetes 对象。你将看到类似于以下输出的内容：

```
root@cka-control-plane:/# kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.project
➥ calico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.or
➥ g created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.
➥ org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico
➥ .org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org 
➥ created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations
➥ .crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.
➥ projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.
➥ projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
```

现在你已经安装了安装 Calico CNI 所需的 Kubernetes 对象，你可以使用命令`kubectl get no`来验证节点是否处于就绪状态。你将看到类似于以下内容的输出：

```
root@cka-control-plane:/# kubectl get no
NAME                STATUS   ROLES           AGE   VERSION
cka-control-plane   Ready    control-plane   61m   v1.25.0-beta.0
cka-worker          Ready    <none>          61m   v1.25.0-beta.0
```

你还可以通过使用命令`kubectl get po -A`看到，CoreDNS Pods 以及 Calico Pods 现在已经在`kube-system`命名空间中启动并运行。输出将类似于以下内容：

```
root@cka-control-plane:/# kubectl get po -A
NAMESPACE            NAME                                        READY   
➥ STATUS
kube-system          calico-kube-controllers-58dbc876ff-l9w9t    1/1     
➥ Running
kube-system          calico-node-g5h7s                           1/1     
➥ Running
kube-system          calico-node-j8g9r                           1/1     
➥ Running
kube-system          coredns-565d847f94-b6jv4                    1/1     
➥ Running
kube-system          coredns-565d847f94-mb554                    1/1     
➥ Running
kube-system          etcd-cka-control-plane                      1/1     
➥ Running
kube-system          kube-apiserver-cka-control-plane            1/1     
➥ Running
kube-system          kube-controller-manager-cka-control-plane   1/1     
➥ Running
kube-system          kube-proxy-9ss5r                            1/1     
➥ Running
kube-system          kube-proxy-dlp2x                            1/1     
➥ Running
kube-system          kube-scheduler-cka-control-plane            1/1     
➥ Running
local-path-storage   local-path-provisioner-684f458cdd-vbskp     1/1     
➥ Running
```

这就完成了 Calico CNI 的安装。
