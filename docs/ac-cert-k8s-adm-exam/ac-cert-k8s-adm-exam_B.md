# 附录 B. 为 kind 集群设置上下文

本附录向您展示如何在具有多个 kubeconfig 文件的 Kubernetes 集群中设置上下文。您将学习如何使用 `kubectl config` 确定您当前所在的上下文以及如何切换到不同的上下文，这应该有助于您在 CKA 考试中舒适地访问多个集群。

## B.1 使用 kubeconfig 设置上下文

正如您在第三章中学到的，您可以有多个与不同集群相关的 kubeconfig 文件，或者您可以将所有集群访问信息合并到一个 kubeconfig 文件中。您现在可以阅读单个 kubeconfig 文件及其内容。此 kubeconfig 文件（命名为 `admin.conf`）位于 `/etc/kubernetes/` 目录中。在引导过程中，该文件被复制，重命名为 `config` 并放置在 `~/.kube/` 目录中。为什么是这样呢？因为 `/etc` 是一个系统目录，需要 root 权限才能访问。由于您是以普通用户（而非 root）运行 `kubectl` 命令，将其复制到您的家目录允许对文件拥有完全所有权。只需运行命令 `kind get kubeconfig --name "kind"` 就可以查看该 kubeconfig 文件的内容（`~/.kube/config`）。同样，您可以使用命令 `kubectl config view --minify` 来查看 kubeconfig 文件的内容。

或者，您可以使用位于不同位置的 kubeconfig 文件，该文件可以通过 `KUBECONFIG` 环境变量进行设置。如果您再次进入 `kind-control-plane` 容器中的 shell，您会看到环境变量已经设置。您可以使用命令 `echo $KUBECONFIG` 来输出环境变量：

```
$ docker exec -it kind-control-plane bash
root@kind-control-plane:/# echo $KUBECONFIG
/etc/kubernetes/admin.conf
```

假设您有一个名为 `config2` 的附加 kubeconfig 文件。此文件位于 `~/Downloads` 目录中，但您仍然希望 `kubectl` 使用它来访问您的 Kubernetes 集群。您可以通过输入 `export KUBECONFIG=~/Downloads/config2` 来告诉 `kubectl` 使用该 kubeconfig 文件对集群进行身份验证，从那时起，它将使用该 `config2` kubeconfig 文件访问集群。如果您想使用两个不同的 kubeconfig 文件，这两个文件位于不同的目录中，您可以输入 `export KUBECONFIG=~/.kube/config:~/Downloads/config2` 并使用它们。要访问每个集群，您可以通过使用命令 `kubectl config use-context` 来切换上下文：

```
$ export KUBECONFIG=~/Downloads/config2:~/.kube/config
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
*         k8s              k8s              k8s
          kind-kind        kind-kind        kind-kind
$ kubectl config use-context kind-kind
Switched to context "kind-kind".
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
          k8s              k8s              k8s
*         kind-kind        kind-kind        kind-kind
```

您需要知道如何切换上下文以进行考试；然而，考试说明将告诉您如何操作。考试中将有多个上下文，因为每个问题都包含在多个集群上执行的一个或多个任务。每次考试提示您进行操作以完成给定任务时，您都需要切换到新的上下文。

## B.2 为 kubectl 设置别名

你可以为 `kubectl` 设置一个缩写名称，因为反复输入 `kubectl` 可能会耗费时间。这被称为别名，最常用的别名是 `k`。例如，设置别名后，你可以输入 `k get no` 而不是 `kubectl get no`。要设置别名，请在控制平面 Bash shell 中输入命令 `alias k=kubectl`（例如，`docker exec -it kind-control-plane bash`）。然后输入命令 `k get no` 以验证别名是否设置正确。输出应类似于以下内容：

```
root@kind-control-plane:/# k get no
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   23h   v1.25.0-beta.0
```

如果你想在退出 shell 并重新进入后设置保持设置，你可以将其添加到主目录中的 .bashrc 文件中。一种简单的方法是在 Bash shell 中输入命令 `echo 'alias k=kubectl' >> ~/.bashrc` 到控制平面节点。然后输入 `source ~/.bashrc` 以立即实施此设置，或者简单地注销并重新登录。输出应类似于以下内容：

```
root@cka-control-plane:/# echo 'alias k=kubectl' >> ~/.bashrc
root@cka-control-plane:/# source ~/.bashrc
root@cka-control-plane:/# k get no
NAME                STATUS   ROLES           AGE   VERSION
cka-control-plane   Ready    control-plane   19m   v1.25.0-beta.0
```

## B.3 设置 kubectl 自动完成

在考试中设置自动完成功能非常重要，因为 Kubernetes 资源包含复杂名称，容易出错。尽可能使用复制粘贴，并在使用 `kubectl` 导航命名空间和资源名称时使用自动完成功能。例如，输入 `k -n c10` 然后按 Tab 键将自动完成命名空间名称为 `k -n c103832034`。使用自动完成功能可以在考试中节省时间，并有助于防止出错。

在 kind 中设置自动完成参数的问题在于按照以下顺序运行以下命令：

```
apt update && apt install -y bash-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

在命令行中逐个运行这些命令，在控制平面节点（即，`docker exec -it kind-control-plane bash`）的 shell 中。
