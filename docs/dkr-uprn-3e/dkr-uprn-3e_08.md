# 第七章：调试容器

一旦你将应用程序部署到生产环境，总会有一天它不如预期般工作。提前了解那一天到来时可以期待什么总是很好的。在继续进行更复杂的部署之前，了解调试容器的基础知识也是很重要的。没有调试技能，将很难看出编排系统出了什么问题。因此，让我们来看看调试容器的内容。

总的来说，调试容器化应用程序与调试系统上的普通进程并没有太大的区别，只是工具有些不同而已。Docker 提供了一些非常好的工具来帮助你！其中一些工具映射到常规系统工具，而一些工具则更进一步。

还很重要的一点是要理解，你的应用程序并不是在与其他 Docker 进程分离的系统中运行。它们共享一个内核，并且根据容器的配置，它们可能会共享其他内容，如存储子系统和网络接口。这意味着你可以从系统中获取关于容器正在执行的操作的大量信息。

如果你习惯于在虚拟机环境中调试应用程序，你可能会认为需要进入容器来检查应用程序的内存或 CPU 使用情况，或者调试其系统调用。但事实并非如此！尽管在许多方面感觉像虚拟化层，但容器中的进程只是 Linux 主机本身上的进程。如果你想查看机器上所有 Linux 容器的进程列表，你可以登录服务器并以你喜欢的命令行选项运行`ps`命令。但是，你可以从任何地方使用`docker container top`命令来查看正在运行的容器中的进程列表，从底层 Linux 内核的视角来看。让我们更详细地看看在调试容器化应用程序时可以做的一些事情，这些事情不需要使用`docker container exec`或`nsenter`。

# 进程输出

在调试容器时，你首先想知道的是容器内部正在运行什么。正如我们之前提到的，Docker 有一个内置命令可以做到这一点：`docker container top`。这不是查看容器内部情况的唯一方法，但是是最简单的方法。让我们看看它是如何工作的：

```
$ docker container run --rm -d --name nginx-debug --rm nginx:latest
796b282bfed33a4ec864a32804ccf5cbbee688b5305f094c6fbaf20009ac2364

$ docker container top nginx-debug

UID   PID  PPID C STIME TTY TIME  CMD
root  2027 2002 0 12:35 ?   00:00 nginx: master process nginx -g daemon off;
uuidd 2085 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2086 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2087 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2088 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2089 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2090 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2091 2027 0 12:35 ?   00:00 nginx: worker process
uuidd 2092 2027 0 12:35 ?   00:00 nginx: worker process

$ docker container stop nginx-debug
```

要运行`docker container top`命令，我们需要向其传递容器的名称或 ID，然后我们会得到一个很好的列表，显示容器内部正在运行的内容，按 PID 排序，就像我们从 Linux 的`ps`命令输出所期望的那样。

但这里有一些奇怪之处。主要的一个是用户 ID 和文件系统的命名空间。

重要的是要理解，特定用户 ID（UID）的用户名在每个容器和主机系统之间可能完全不同。甚至可能会出现某个特定 UID 在容器或主机的*/etc/passwd*文件中没有任何关联的情况。这是因为 Unix 不要求 UID 必须与命名用户关联，而我们在“命名空间”中详细讨论的 Linux 命名空间提供了一些容器对有效用户的概念与基础主机之间的隔离。

让我们看一个更具体的例子。考虑一个运行 Ubuntu 22.04 的生产 Docker 服务器以及其中运行一个内部有 Ubuntu 发行版的容器。如果你在 Ubuntu 主机上运行以下命令，你会看到 UID 7 被命名为`lp`：

```
$ id 7

uid=7(lp) gid=7(lp) groups=7(lp)
```

###### 注意

这里使用的 UID 号并没有什么特别之处。你不需要特别注意它。之所以选择它，只是因为它在两个平台上默认使用，但代表着不同的用户名。

如果我们然后进入该 Docker 主机上的标准 Fedora 容器，你会看到 UID 7 在*/etc/passwd*中被设置为`halt`。通过运行以下命令，您可以看到容器对 UID 7 有完全不同的看法：

```
$ docker container run --rm -it fedora:latest /bin/bash

root@c399cb807eb7:/# id 7
uid=7(halt) gid=0(root) groups=0(root)

root@c399cb807eb7:/# grep x:7: /etc/passwd
halt:x:7:0:halt:/sbin:/sbin/halt

root@409c2a8216b1:/# exit
```

如果我们在理论上的 Ubuntu Docker 服务器上以 UID 7（`-u 7`）运行`ps aux`，我们会看到 Docker 主机显示容器进程由`lp`而不是`halt`运行：

```
$ docker container run --rm -d -u 7 fedora:latest sleep 120

55…c6

$ ps aux | grep sleep

lp          2388  0.2  0.0   2204   784 ?     … 0:00 sleep 120
vagrant     2419  0.0  0.0   5892  1980 pts/0 … 0:00 grep --color=auto sleep
```

这可能特别令人困惑，特别是如果像`nagios`或`postgres`这样的知名用户在主机系统上配置好了，但在容器中没有配置，而容器却使用相同的 ID 运行其进程。这种命名空间可能会导致`ps`命令的输出看起来非常奇怪。例如，如果你不仔细观察，可能会看到你的 Docker 主机上的`nagios`用户正在运行容器内启动的`postgresql`守护进程。

###### 提示

其中一个解决方案是为您的容器分配一个非零的 UID。在您的 Docker 服务器上，您可以创建一个 UID 为 5000 的`container`用户，然后在基础容器镜像中创建相同的用户。如果您然后将所有容器作为 UID 5000（`-u 5000`）运行，不仅可以通过在 Docker 主机上显示所有运行容器进程的`container`用户来提高系统安全性，而且还可以使`ps`命令的输出更易于解析。一些系统使用`nobody`或`daemon`用户来达到相同的目的，但我们更喜欢`container`以提升清晰度。关于这种工作方式的更多细节可以在“命名空间”中找到。

同样，由于进程对文件系统有不同的视图，`ps`输出中显示的路径是相对于容器而不是主机的。在这些情况下，知道它在容器中是一大优势。

这就是你使用 Docker 工具来查看容器中正在运行的内容的方法。但这并不是唯一的方式，在调试情况下，这可能不是最佳方式。如果你进入 Docker 服务器并运行普通的 Linux `ps` 命令来查看正在运行的内容，你会得到一个包含所有容器化和非容器化内容的完整列表，就像它们都是等效进程一样。有一些方法可以查看进程输出以使事情更加清晰。例如，你可以通过查看 Linux `ps` 输出的树形式来促进调试，以便你可以看到所有从 Docker 派生的进程。当你使用 BSD 命令行标志查看当前运行两个容器的系统时，这可能是你会看到的样子；我们将只截取我们关心的部分输出。

###### 注意

Docker Desktop 的虚拟机包含大多数 Linux 工具的最小版本，其中一些命令的输出可能与将标准 Linux 服务器用作 Docker 守护程序主机时获得的输出不同。

```
$ ps axlfww

… /usr/bin/containerd
…
… /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
… \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 \
 -container-ip 172.17.0.2 -container-port 8080
… \_ /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 8080 \
 -container-ip 172.17.0.2 -container-port 8080
…
… /usr/bin/containerd-shim-runc-v2 -namespace moby -id 97…3d -address /run/…
… \_ sleep 120
…
… /usr/bin/containerd-shim-runc-v2 -namespace moby -id 69…7c -address /run/…
```

###### 注意

在本示例中，许多 `ps` 命令仅适用于具有完整 `ps` 命令的 Linux 发行版。某些精简版的 Linux，如 Alpine，运行 BusyBox shell，它不支持完整的 `ps` 并且不会显示某些输出。我们建议在主机系统上运行类似 Ubuntu 或 Fedora CoreOS 的完整发行版。

在这里，你可以看到我们正在运行一个 `containerd` 实例，它是 Docker 守护程序使用的主要容器运行时。`dockerd` 目前有两个 `docker-proxy` 子进程运行，我们将在 “网络检查” 中详细讨论它们。

每个使用 `containerd-shim-runc-v2` 的进程表示一个单独的容器以及在该容器中运行的所有进程。在这个例子中，我们有两个容器。它们显示为 `containerd-shim-runc-v2`，后面跟着有关该进程的一些额外信息，包括容器 ID。在这种情况下，我们正在一个实例中运行 Google 的 `cadvisor` 和另一个容器中的 `sleep`。每个映射端口的容器将至少有一个 `docker-proxy` 进程，用于在容器和主机 Docker 服务器之间映射所需的网络端口。在本示例中，两个 `docker-proxy` 进程都与 `cadvisor` 相关联。其中一个映射 IPv4 地址的端口，另一个映射 IPv6 地址的端口。

由于 `ps` 的树形输出，很明显可以看出哪些进程运行在哪些容器中。如果你更喜欢 Unix SysV 命令行标志，可以使用 `ps -ejH` 获得类似但不够美观的树形输出：

```
$ ps -ejH

… containerd
…
… dockerd
…   docker-proxy
…   docker-proxy
…
… containerd-shim
…   cadvisor
…
… containerd-shim
…   sleep
```

通过使用 `pstree` 命令，可以更简洁地查看 `docker` 进程树。在这里，我们将使用 `pidof` 将其范围限定为属于 `docker` 的树：

```
$ pstree `pidof dockerd`

dockerd─┬─docker-proxy───7*[{docker-proxy}]
 ├─docker-proxy───6*[{docker-proxy}]
 └─10*[{dockerd}]
```

这不会显示 PIDs，因此只能用于了解连接方式。但是，当主机上运行大量进程时，这个概念上清晰的输出会更加简洁，提供了一个很好的高级地图，展示了事物如何连接。在这里，我们可以看到与先前的`ps`输出中显示的相同的容器，但是树被折叠了，所以当有七个重复的进程时会得到`7*`这样的倍增器。

如果我们运行`pstree`，我们可以得到一个带有 PIDs 的完整树，如下所示：

```
$ pstree -p `pidof dockerd`

dockerd(866)─┬─docker-proxy(3050)─┬─{docker-proxy}(3051)
 │                    ├─{docker-proxy}(3052)
 │                    ├─{docker-proxy}(3053)
 │                    ├─{docker-proxy}(3054)
 │                    ├─{docker-proxy}(3056)
 │                    ├─{docker-proxy}(3057)
 │                    └─{docker-proxy}(3058)
 ├─docker-proxy(3055)─┬─{docker-proxy}(3059)
 │                    ├─{docker-proxy}(3060)
 │                    ├─{docker-proxy}(3061)
 │                    ├─{docker-proxy}(3062)
 │                    ├─{docker-proxy}(3063)
 │                    └─{docker-proxy}(3064)
 ├─{dockerd}(904)
 ├─{dockerd}(912)
 ├─{dockerd}(913)
 ├─{dockerd}(914)
 ├─{dockerd}(990)
 ├─{dockerd}(1014)
 ├─{dockerd}(1066)
 ├─{dockerd}(1605)
 ├─{dockerd}(1611)
 └─{dockerd}(2228)
```

此输出为我们提供了一个非常好的视角，查看 Docker 上所有的进程及其运行情况。

如果你想要检查单个容器及其进程，你可以确定容器的主进程 ID，然后使用`pstree`来查看所有相关的子进程：

```
$ ps aux | grep containerd-shim-runc-v2
root    3072  … /usr/bin/containerd-shim-runc-v2 -namespace moby -id 69…7c …
root    4489  … /usr/bin/containerd-shim-runc-v2 -namespace moby -id f1…46 …
vagrant 4651  … grep --color=auto shim

$ pstree -p 3072
containerd-shim(3072)─┬─cadvisor(3092)─┬─{cadvisor}(3123)
 │                ├─{cadvisor}(3124)
 │                ├─{cadvisor}(3125)
 │                ├─{cadvisor}(3126)
 │                ├─{cadvisor}(3127)
 │                ├─{cadvisor}(3128)
 │                ├─{cadvisor}(3180)
 │                ├─{cadvisor}(3181)
 │                └─{cadvisor}(3182)
 ├─{containerd-shim}(3073)
 ├─{containerd-shim}(3074)
 ├─{containerd-shim}(3075)
 ├─{containerd-shim}(3076)
 ├─{containerd-shim}(3077)
 ├─{containerd-shim}(3078)
 ├─{containerd-shim}(3079)
 ├─{containerd-shim}(3080)
 ├─{containerd-shim}(3121)
 └─{containerd-shim}(3267)
```

# 进程检查

如果你登录到 Docker 服务器，你可以使用所有标准的调试工具来检查运行中的进程。像`strace`这样的常见调试工具按预期工作。在下面的代码中，我们将检查运行在容器中的`nginx`进程：

```
$ docker container run --rm -d --name nginx-debug --rm nginx:latest

$ docker container top nginx-debug

UID      PID   PPID  … CMD
root     22983 22954 … nginx: master process nginx -g daemon off;
systemd+ 23032 22983 … nginx: worker process
systemd+ 23033 22983 … nginx: worker process

$ sudo strace -p 23032

strace: Process 23032 attached
epoll_pwait(10,
```

###### 警告

如果你运行`strace`，你需要输入 Ctrl-C 来退出`strace`进程。

你可以看到，我们得到的输出与主机上非容器化进程的输出相同。同样，`lsof`显示了进程中打开的文件和套接字的预期工作方式：

```
$ sudo lsof -p 22983
COMMAND   PID USER … NAME
nginx   22983 root … /
nginx   22983 root … /
nginx   22983 root … /usr/sbin/nginx
nginx   22983 root … /usr/sbin/nginx (stat: No such file or directory)
nginx   22983 root … /lib/aarch64-linux-gnu/libnss_files-2.31.so (stat: …
nginx   22983 root … /lib/aarch64-linux-gnu/libc-2.31.so (stat: …
nginx   22983 root … /lib/aarch64-linux-gnu/libz.so.1.2.11 (path inode=…)
nginx   22983 root … /usr/lib/aarch64-linux-gnu/libcrypto.so.1.1 (stat: …
nginx   22983 root … /usr/lib/aarch64-linux-gnu/libssl.so.1.1 (stat: …
nginx   22983 root … /usr/lib/aarch64-linux-gnu/libpcre2-8.so.0.10.1 (stat: …
nginx   22983 root … /lib/aarch64-linux-gnu/libcrypt.so.1.1.0 (path …
nginx   22983 root … /lib/aarch64-linux-gnu/libpthread-2.31.so (stat: …
nginx   22983 root … /lib/aarch64-linux-gnu/libdl-2.31.so (stat: …
nginx   22983 root … /lib/aarch64-linux-gnu/ld-2.31.so (stat: …
nginx   22983 root … /dev/zero
nginx   22983 root … /dev/null
nginx   22983 root … pipe
nginx   22983 root … pipe
nginx   22983 root … pipe
nginx   22983 root … protocol: UNIX-STREAM
nginx   22983 root … pipe
nginx   22983 root … pipe
nginx   22983 root … protocol: TCP
nginx   22983 root … protocol: TCPv6
nginx   22983 root … protocol: UNIX-STREAM
nginx   22983 root … protocol: UNIX-STREAM
nginx   22983 root … protocol: UNIX-STREAM
```

请注意，文件的路径都是相对于容器对后备文件系统的视图，这与主机视图不同。因此，如果你在主机系统上，可能无法轻松地找到正在运行的容器中的特定文件。在大多数情况下，最好使用`docker container exec`进入容器，以便使用与其中进程具有相同视图的方式查看文件。

只要你是`root`并且有适当的权限，就可以像这样运行 GNU 调试器（`gdb`）和其他进程检查工具。

值得一提的是，也可以运行一个新的调试容器，可以查看现有容器的进程，从而提供额外的工具来调试问题。我们稍后将在“命名空间”和“安全性”中讨论该命令的底层细节。

```
$ docker container run -ti --rm --cap-add=SYS_PTRACE \
    --pid=container:nginx-debug spkane/train-os:latest bash

[root@e4b5d2f3a3a7 /]# ps aux
USER PID %CPU %MEM … TIME COMMAND
root   1  0.0  0.2 … 0:00 nginx: master process nginx -g daemon off;
101   30  0.0  0.1 … 0:00 nginx: worker process
101   31  0.0  0.1 … 0:00 nginx: worker process
root 136  0.0  0.1 … 0:00 bash
root 152  0.0  0.2 … 0:00 ps aux

[root@e4b5d2f3a3a7 /]# strace -p 1
strace: Process 1 attached
rt_sigsuspend([], 8

[Control-C]
strace: Process 1 detached
<detached …>

[root@e4b5d2f3a3a7 /]# exit

$ docker container stop nginx-debug
```

###### 警告

你需要输入 Ctrl-C 来退出`strace`进程。

# 控制进程

当你直接在 Docker 服务器上有一个 shell 时，你可以通过多种方式处理容器化的进程，就像处理系统上运行的任何其他进程一样。如果你是远程的，你可能会使用`docker container kill`发送信号，因为这样做更方便。但是如果你已经登录到一个 Docker 服务器进行调试会话，或者因为 Docker 守护进程没有响应，你可以像处理任何其他进程一样直接`kill`这个进程。

除非您终止容器中顶级进程（容器内部的 PID 1），否则终止一个进程不会终止容器本身。如果终止一个失控的进程可能是有意义的，但这可能会使容器处于意外的状态。开发人员可能期望看到他们的容器在 `docker container ls` 中列出所有进程都在运行。这也可能会让像 Mesos 或 Kubernetes 这样的调度器或其他健康检查您应用程序的系统感到困惑。请记住，容器对外界表现为一个单一的整体。如果需要终止容器内部的某些内容，最好是替换整个容器。容器提供了一个工具可以进行交互的抽象。它们期望容器内部是可预测和一致的。

终止进程并不是发送信号的唯一原因。由于容器化的进程在许多方面都像普通进程一样，因此它们可以接收到 Linux `kill` 命令手册中列出的整套 Unix 信号。许多 Unix 程序在接收到特定的预定义信号时会执行特殊操作。例如，当接收到 `SIGUSR1` 信号时，`nginx` 会重新打开其日志。使用 Linux `kill` 命令，您可以向本地服务器上的容器进程发送任何 Unix 信号。

因为容器的工作方式与其他进程类似，所以了解它们如何与您的应用程序进行交互是非常重要的，特别是在一些会生成后台子进程的情况下，即任何进行分叉和守护化以使父进程不再管理子进程生命周期的情况。Jenkins 构建容器就是人们常见的一个例子，他们在这种情况下可能会出现问题。当守护进程分叉到后台时，它们会成为 Unix 系统上 PID 1 的子进程。进程 1 是特殊的，通常是某种 `init` 进程。

PID 1 负责确保子进程被回收。在您的容器中，默认情况下，您的主进程将是 PID 1。由于您可能不会处理应用程序中子进程的回收，因此可能会在容器中产生僵尸进程。对于这个问题有几种解决方案。首先是在容器中运行一个您自己选择的 init 系统，这个系统能够处理 PID 1 的责任。像 `s6`、`runit` 以及前面笔记中描述的其他系统可以很容易地在容器内部使用。

但是 Docker 本身提供了一个更简单的选项，仅解决这一个情况而不需要具备完整 init 系统的所有功能。如果您在 `docker container run` 命令中提供 `--init` 标志，Docker 将启动一个非常小的 init 进程，基于 [`tini` 项目](https://github.com/krallin/tini)，该进程将在容器启动时作为 PID 1 运行。无论您在 *Dockerfile* 中指定的 `CMD` 是什么，都将传递给 `tini` 并且工作方式与您期望的相同。然而，它会取代您在 *Dockerfile* 的 `ENTRYPOINT` 部分可能设置的任何内容。

如果您在启动 Linux 容器时没有使用 `--init` 标志，您的进程列表中会得到类似这样的内容：

```
$ docker container run --rm -it alpine:3.16 sh
/ # ps -ef

PID   USER     TIME   COMMAND
 1 root       0:00 sh
 5 root       0:00 ps -ef

/ # exit
```

注意，在这种情况下，我们启动的 `CMD` 是 PID 1。这意味着它负责子进程的收割。如果我们启动的容器中这一点很重要，我们可以传递 `--init` 来确保当父进程退出时，子进程被收割：

```
$ docker container run --rm -it --init alpine:3.16 sh
/ # ps -ef

PID   USER     TIME   COMMAND
 1 root       0:00 /sbin/docker-init -- sh
 5 root       0:00 sh
 6 root       0:00 ps -ef

/ # exit
```

这里，您可以看到 PID 1 进程是 `/sbin/docker-init`。这反过来启动了作为命令行指定的 shell 二进制文件。因为现在容器内部有一个 init 系统，PID 1 的责任落在它身上，而不是我们用来调用容器的命令。在大多数情况下，这是您想要的。您可能不需要一个 init 系统，但是在生产环境中至少考虑在您的容器内部包含 `tini`。

一般来说，如果您在容器内运行多个父进程或者有不正确响应 Unix 信号的进程，可能只有在容器内部需要一个 init 进程。

# 网络检查

与进程检查相比，调试容器化应用程序在网络层面可能更加复杂。与运行在主机上的传统进程不同，Linux 容器可以通过多种方式连接到网络。如果您正在运行默认设置，就像绝大多数人一样，那么您的所有容器都通过 Docker 创建的默认桥接网络连接到网络。这是一个虚拟网络，其中主机是通往世界其他地方的网关。我们可以使用 Docker 提供的工具检查这些虚拟网络。您可以通过调用 `docker network ls` 命令来显示存在的网络：

```
$ docker network ls

NETWORK ID     NAME      DRIVER    SCOPE
f9685b50d57c   bridge    bridge    local
8acae1680cbd   host      host      local
fb70d67499d3   none      null      local
```

这里我们可以看到默认的桥接网络、主机网络，用于任何以 `host` 网络模式运行的容器（参见 “主机网络”），以及禁用容器完全的 none 网络。如果您使用 `docker compose` 或其他编排工具，它们可能会创建额外的网络，并使用不同的名称。

但是看到存在哪些网络并不能使我们更容易地看到这些网络上存在什么。因此，您可以使用`docker network inspect`命令查看连接到任何特定命名网络的容器。这会产生相当多的输出。它会显示连接到指定网络的所有容器以及有关网络本身的许多详细信息。让我们来看看默认桥接网络：

```
$ docker network inspect bridge
```

```
[
    {
        "Name": "bridge",
        …
        "Driver": "bridge",
        "EnableIPv6": false,
        …
        "Containers": {
            "69e9…c87c": {
                "Name": "cadvisor",
                …
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "a2a8…e163": {
                "Name": "nginx-debug",
                …
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            …
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            …
        },
        "Labels": {}
    }
]
```

为了减少输出量，我们在这里省略了一些细节。但是我们可以看到的是，在桥接网络上有两个容器，它们附加到主机上的`docker0`桥接上。我们还可以看到每个容器的 IP 地址（`IPv4Address`和`IPv6Address`）以及它们绑定到的主机网络地址（`host_binding_ipv4`）。这在您试图理解桥接网络的内部结构时非常有用。如果您的容器位于不同的网络上，根据网络的配置方式，它们可能无法相互连通。

###### 提示

一般情况下，我们建议保留容器在默认桥接网络上，直到有充分理由或者运行`docker compose`或者自动管理容器网络的调度程序为止。此外，以某种可识别的方式命名您的容器在这里也是有帮助的，因为我们无法看到镜像信息。在这个输出中，名称和 ID 是我们唯一能够在`docker container ls`列表中找到的参考信息。一些调度程序在命名容器方面做得不好，这真是太糟糕了，因为这对调试非常有帮助。

正如我们所见，容器通常会有自己的网络堆栈和 IP 地址，除非它们在主机网络模式下运行，我们将在“网络”中进一步讨论这一点。但是当我们从主机机器自身查看它们时会怎么样？因为容器有自己的网络和地址，它们不会出现在主机上所有`netstat`输出中。但我们知道，您映射到容器的端口是绑定到主机的。

在 Docker 服务器上运行`netstat -an`正常工作，如下所示：

```
$ sudo netstat -an

Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address        State
tcp        0      0 0.0.0.0:8080            0.0.0.0:*              LISTEN
tcp        0      0 127.0.0.53:53           0.0.0.0:*              LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*              LISTEN
tcp        0      0 192.168.15.158:22       192.168.15.120:63920   ESTABLISHED
tcp6       0      0 :::8080                 :::*                   LISTEN
tcp6       0      0 :::22                   :::*                   LISTEN
udp        0      0 127.0.0.53:53           0.0.0.0:*
udp        0      0 192.168.15.158:68       0.0.0.0:*
raw6       0      0 :::58                   :::*                   7
…
```

在这里，我们可以看到我们正在监听的所有接口。我们的容器绑定到 IP 地址`0.0.0.0`的端口`8080`上。这显示出来了。但是当我们要求`netstat`显示绑定到端口的进程名称时会发生什么？

```
$ sudo netstat -anp

Active Internet connections (servers and established)
Proto  … Local Address           Foreign Address      … PID/Program name
tcp    … 0.0.0.0:8080            0.0.0.0:*            … 1516/docker-proxy
tcp    … 127.0.0.53:53           0.0.0.0:*            … 692/systemd-resolve
tcp    … 0.0.0.0:22              0.0.0.0:*            … 780/sshd: /usr/sbin
tcp    … 192.168.15.158:22       192.168.15.120:63920 … 1348/sshd: vagrant
tcp6   … :::8080                 :::*                 … 1522/docker-proxy
tcp6   … :::22                   :::*                 … 780/sshd: /usr/sbin
udp    … 127.0.0.53:53           0.0.0.0:*            … 692/systemd-resolve
udp    … 192.168.15.158:68       0.0.0.0:*            … 690/systemd-network
raw6   … :::58                   :::*                 … 690/systemd-network
```

我们看到相同的输出，但注意绑定到端口的是什么：`docker-proxy`。这是因为在其默认配置中，Docker 有一个用 Go 编写的代理，位于所有容器和外部世界之间。这意味着当我们查看这个输出时，所有通过 Docker 运行的容器都将与`docker-proxy`关联。请注意，这里没有任何线索表明`docker-proxy`正在处理哪个特定的容器。幸运的是，`docker container ls`显示了绑定到哪些端口的容器，所以这并不是什么大问题。但这并不明显，在你调试生产故障之前，你可能希望意识到这一点。然而，通过向`netstat`传递`p`标志有助于识别与容器关联的端口。

###### 注意

如果你在容器中使用主机网络，则会跳过此层。没有`docker-proxy`，容器中的进程可以直接绑定到端口。它还显示为`netstat -anp`输出中的正常进程。

其他网络检查命令的工作大部分如预期，包括`tcpdump`，但重要的是要记住`docker-proxy`在这里，在主机的网络接口和容器之间，并且容器在虚拟网络上有它们自己的网络接口。

# 镜像历史

当你构建和部署单个容器时，很容易追踪它的来源以及它位于哪些镜像之上。但是当你运送许多由不同团队构建和维护的镜像的容器时，这很快变得难以管理。你怎么知道哪些层实际上是你的容器运行在其上面的？你的容器的镜像标签有希望清楚地表明你正在运行的应用程序的哪个构建，但是镜像标签并不透露任何关于构建你的应用程序的镜像层的信息。`docker image history`正是为此而设。你可以看到检查的镜像中存在的每一层，每一层的大小以及用于构建它们的命令：

```
$ docker image history redis:latest

IMAGE        … CREATED BY                                      SIZE    COMMENT
e800a8da9469 … /bin/sh -c #(nop)  CMD ["redis-server"]         0B
<missing>    … /bin/sh -c #(nop)  EXPOSE 6379                  0B
<missing>    … /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B
<missing>    … /bin/sh -c #(nop) COPY file:e873a0e3c13001b5…   661B
<missing>    … /bin/sh -c #(nop) WORKDIR /data                 0B
<missing>    … /bin/sh -c #(nop)  VOLUME [/data]               0B
<missing>    … /bin/sh -c mkdir /data && chown redis:redis …   0B
<missing>    … /bin/sh -c set -eux;   savedAptMark="$(apt-m…   32.4MB
<missing>    … /bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_SHA=f0…   0B
<missing>    … /bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_URL=ht…   0B
<missing>    … /bin/sh -c #(nop)  ENV REDIS_VERSION=7.0.4      0B
<missing>    … /bin/sh -c set -eux;  savedAptMark="$(apt-ma…   4.06MB
<missing>    … /bin/sh -c #(nop)  ENV GOSU_VERSION=1.14        0B
<missing>    … /bin/sh -c groupadd -r -g 999 redis && usera…   331kB
<missing>    … /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>    … /bin/sh -c #(nop) ADD file:6039adfbca55ed34a…   74.3MB
```

使用`docker image history`可以很有用，例如，当你试图确定最终镜像大小比预期大得多的原因时。层按顺序列出，最底部的是列表的第一层，最顶部的是列表的最后一层。

在一些情况下，我们可以看到命令输出已被截断。对于长命令，向`docker image history`命令添加`--no-trunc`选项将让您看到用于构建每个层的完整命令。只要注意`--no-trunc`会使输出在大多数情况下变得更大且更难以视觉扫描。

# 检查容器

在 第四章 中，我们向您展示了如何查看 `docker container inspect` 输出以查看容器的配置。但在此之下是主机磁盘上专用于容器的目录。通常情况下，这是 */var/lib/docker/containers*。如果查看该目录，其中包含非常长的 SHA 哈希，如下所示：

```
$ sudo ls /var/lib/docker/containers

106ead0d55af55bd803334090664e4bc821c76dadf231e1aab7798d1baa19121
28970c706db0f69716af43527ed926acbd82581e1cef5e4e6ff152fce1b79972
3c4f916619a5dfc420396d823b42e8bd30a2f94ab5b0f42f052357a68a67309b
589f2ad301381b7704c9cade7da6b34046ef69ebe3d6929b9bc24785d7488287
959db1611d632dc27a86efcb66f1c6268d948d6f22e81e2a22a57610b5070b4d
a1e15f197ea0996d31f69c332f2b14e18b727e53735133a230d54657ac6aa5dd
bad35aac3f503121abf0e543e697fcade78f0d30124778915764d85fb10303a7
bc8c72c965ebca7db9a2b816188773a5864aa381b81c3073b9d3e52e977c55ba
daa75fb108a33793a3f8fcef7ba65589e124af66bc52c4a070f645fffbbc498e
e2ac800b58c4c72e240b90068402b7d4734a7dd03402ee2bce3248cc6f44d676
e8085ebc102b5f51c13cc5c257acb2274e7f8d1645af7baad0cb6fe8eef36e24
f8e46faa3303d93fc424e289d09b4ffba1fc7782b9878456e0fe11f1f6814e4b
```

这有点令人生畏。但这些只是容器长形式的容器 ID。如果要查看特定容器的配置，只需使用 `docker container ls` 查找其短 ID，然后找到匹配的目录即可：

```
$ docker container ls

CONTAINER ID   IMAGE                                   COMMAND              …
c58bfeffb9e6   gcr.io/cadvisor/cadvisor:v0.44.1-test   "/usr/bin/cadvisor…" …
```

你可以从 `docker container ls` 中查看短 ID，然后将其与 `ls /var/lib/docker/containers` 输出匹配，查看以 `c58bfeffb9e6` 开头的目录。命令行的选项卡完成在这里非常有帮助。如果需要精确匹配，可以执行 `docker container inspect c58bfeffb9e6` 并从输出中获取长 ID。此目录包含与容器相关的一些非常有趣的文件：

```
$ cd /var/lib/docker/containers/\
c58bfeffb9e6e607f3aacb4a06ca473535bf9588450f08be46baa230ab43f1d6

$ ls -la

total 48
drwx--x---  4 root root 4096 Aug 20 10:38 .
drwx--x--- 30 root root 4096 Aug 20 10:25 ..
-rw-r-----  1 root root  635 Aug 20 10:34 c58bf…f1d6-json.log
drwx------  2 root root 4096 Aug 20 10:24 checkpoints
-rw-------  1 root root 4897 Aug 20 10:38 config.v2.json
-rw-r--r--  1 root root 1498 Aug 20 10:38 hostconfig.json
-rw-r--r--  1 root root   13 Aug 20 10:24 hostname
-rw-r--r--  1 root root  174 Aug 20 10:24 hosts
drwx--x---  2 root root 4096 Aug 20 10:24 mounts
-rw-r--r--  1 root root  882 Aug 20 10:24 resolv.conf
-rw-r--r--  1 root root   71 Aug 20 10:24 resolv.conf.hash
```

正如我们在 第五章 中讨论的，此目录包含一些直接绑定到容器中的文件，如 *hosts*、*resolv.conf* 和 *hostname*。如果您正在运行默认的日志记录机制，那么此目录也是 Docker 存储显示在 `docker container logs` 命令中的日志的 JSON 文件的地方，支持 `docker container inspect` 输出的 JSON 配置文件 (*config.v2.json*)，以及容器的网络配置 (*hostconfig.json*)。*resolv.conf.hash* 文件由 Docker 使用，用于确定容器的文件与主机上当前文件有何不同，以便进行更新。

即使我们无法进入容器，或者 `docker` 不响应时，此目录在严重故障事件中也非常有帮助。了解容器的配置方式也非常有用。请记住，修改这些文件并不是一个好主意。Docker 期望它们包含现实情况，如果您改变这种现实，会带来麻烦。但这是了解容器内发生的事情的另一途径。

# 文件系统检查

无论实际使用的后端是什么，Docker 都具有分层文件系统，允许跟踪任何给定容器中的更改。这就是在构建时如何组装镜像的方式，但当您试图确定 Linux 容器是否有任何更改时，它也很有用。容器化应用的一个常见问题是它们可能会继续向容器文件系统写入东西。通常情况下，您不希望您的容器这样做，尽可能地，并且弄清楚您的进程是否一直在容器中写入东西可以帮助调试。有时这对于找出容器中存在的杂乱日志文件也很有帮助。与大多数核心工具一样，这种检查方式内置于`docker`命令行工具和 API 中。让我们看看这给我们展示了什么。让我们启动一个快速容器，并使用它的名称来探索这个问题：

```
$ docker container run --rm -d --name nginx-fs nginx:latest
1272b950202db25ee030703515f482e9ed576f8e64c926e4e535ba11f7536cc4

$ docker container diff nginx-fs
C /run
A /run/nginx.pid
C /var
C /var/cache
C /var/cache/nginx
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
C /etc
C /etc/nginx
C /etc/nginx/conf.d
C /etc/nginx/conf.d/default.conf

$ docker container stop nginx-fs
nginx-fs
```

每一行都以`A`或`C`开头，分别代表*新增*或*更改*。我们可以看到该容器正在运行`nginx`，`nginx`配置文件已被写入，并且在名为`/var/cache/nginx`的新目录中创建了一些临时文件。当您试图优化和加固容器的文件系统使用时，了解容器文件系统的使用方式非常有用。

进一步详细的检查需要使用`docker container export`、`docker container exec`或`nsenter`等方法来探索容器中的确切内容。但是，`docker container diff`给了您一个很好的起点。

# Wrap-Up

到目前为止，您应该已经对如何在开发和生产中部署和调试单个容器有了很好的了解，但是如何开始为更大的应用程序生态系统扩展呢？在下一章中，我们将介绍一个较简单的 Docker 编排工具：Docker Compose。这个工具是单个 Linux 容器和生产编排系统之间的良好桥梁。它在开发环境中提供了很多价值，并贯穿整个 DevOps 流水线。
