# 11 额外的安全考虑

本章涵盖

+   在不同的独立服务器、不同的虚拟机（VM）和容器内安全运行应用程序

+   通过服务运行容器与通过 fork 和 exec 作为容器引擎子进程运行容器的比较

+   用于将容器彼此隔离的 Linux 安全功能

+   设置容器镜像信任

+   签署镜像并信任它们

在本章中，我回顾并演示了使用 Podman 运行容器时的一些额外的安全考虑。其中一些内容在其他章节中已经介绍过，但我认为从安全角度集中关注这些功能是有用的。

我看到人们在运行容器时最常见的一个问题是，当容器进程被拒绝某些访问时，用户的第一个反应是运行容器在`--privileged`模式下，这将关闭您容器的所有安全隔离。了解如何处理本章中讨论的安全功能可以帮助您避免这种情况。

## 11.1 守护进程与 fork/exec 模型

在前面的章节中，您已经学到了很多关于像 Docker 这样的守护进程与 Podman 使用的 fork/exec 模型之间的问题。

### 11.1.1 对 docker.sock 的访问

回想一下，Docker 默认运行一个由 root 用户拥有的守护进程。这意味着任何可以访问守护进程的用户都可以在系统上以完全 root 权限启动进程。Docker 建议一些用户将他们的账户添加到`/etc/group`中的 docker 组。在某些发行版中，这允许您无需 root 权限即可访问`/run/docker.sock`：

```
# ls -l /run/docker.sock
srw-rw----. 1 root docker 0 Jun 13 14:54 /run/docker.sock
```

您可以像运行 Podman 容器一样运行 Docker 容器：

```
$ docker run registry.access.redhat.com/ubi8-micro echo hi
Unable to find image 'registry.access.redhat.com/ubi8-micro:latest' locally|
latest: Pulling from ubi8-micro
4f4fb700ef54: Pull complete
b6d5e0581b2f: Pull complete
Digest: sha256:a519ab06c0287085c352af0d2b84f2a2b257d2afb2e554b8d38a076cd6205b48
Status: Downloaded newer image for registry.access.redhat.com/
ubi8-micro:latest
hi
```

这让许多用户感到兴奋，直到他们意识到他们也可以通过简单的 Docker 命令在他们的系统上启动 root shell：

```
$ docker run -ti --name hack -v /:/host --privileged 
➥ registry.access.redhat.com/ubi8-micro chroot /host
# cat /etc/shadow
...
```

到目前为止，您在主机系统上拥有一个完全特权的 root shell，在其中您可以随意破解机器。不仅如此，Docker 默认将所有日志记录为基于文件的。当您完成破解系统后，您可以删除日志文件和您所有活动的记录：

```
$ docker rm hack
hack
```

使用无根 Podman，您无法这样做，因为当您运行容器时，容器进程是以您的用户 UID 运行的，并且只能访问与您账户中任何进程相同的文件。管理员确定他们是否被黑客攻击的一种方法是通过检查日志系统，包括审计日志。

### 11.1.2 审计和日志记录

Linux 系统的一个关键特性是跟踪当进程在系统上运行时它们做了什么。当您登录到 Linux 系统时，内核会将您的 UID 记录到/proc/self/loginuid 中的进程数据中。您可以通过执行以下命令查看这些数据：

```
$ cat /proc/self/loginuid
3267
```

此第一个进程创建的所有进程在登录后都保持此字段。即使您使用`setuid`程序，如`su`或`sudo`，您的`loginuid`仍然保持不变：

```
$ sudo cat /proc/self/loginuid
3267
```

即使您启动了一个容器，`loginuid` 仍然保持不变。在下一个例子中，您以守护进程模式运行一个简单的容器，然后使用 `podman inspect` 获取睡眠进程的 PID，最后检查容器化进程的 `loginuid`：

```
$ podman run -d ubi8-micro sleep 20
1c55b9cfa0cd20c36da4b606415e190a6c20cc868d3486981c7713d41ee9ea6a
$ podman inspect -l --format '{{ .State.Pid }}'
119394
$ cat /proc/119394/loginuid
3267
```

注意，容器化进程仍在使用您的 `loginuid` 运行。这表明只要容器引擎使用 fork/exec 模型，内核就可以通过此字段跟踪哪个用户在系统上启动了容器进程。如果您使用 Docker 运行相同的测试，您会得到非常不同的结果：

```
$ docker run -d registry.access.redhat.com/ubi8-micro sleep 20
df2302cf8c6385df2b86ccd3429166e0d8dd0c9f0d0139e98e6354809a04080e
$ docker inspect df2302cf8c6 --format '{{ .State.Pid }}'
120022
$ cat /proc/120022/loginuid
4294967295
```

您看到的不是您的 `loginuid`，而是 `4294967295`，这是 2³² – 1。这是 Linux 内核表示 `-1` 的方式，这是系统启动的所有进程的默认 `loginuid`，而不是登录系统的用户启动的进程。原因是 Docker 使用客户端-服务器模型，容器进程是 Docker 守护进程的子进程，而不是 Docker 客户端。由于 Docker 守护进程是在系统启动时由 systemd 启动的，因此所有子进程都具有 `-1` 的 `loginuid`。

内核的审计子系统在完成可审计事件时记录系统上每个进程的 `loginuid`。例如，当用户登录和注销系统时，这些事件会被记录下来。修改 /etc/passwd 和 /etc/shadow 也是可记录的事件。

以下是我今天登录系统时的 `USER_START` 审计日志条目。我的 UID `3267` 被记录，以及我的用户名：

```
# ausearch -m USER_START
type=USER_START msg=audit(1651064687.963:315): pid=2579 uid=0 auid=3267 
➥ ses=3 subj=system_u:system_r:xdm_t:s0-s0:c0.c1023 msg='op=PAM:session_open 
➥ grantors=pam_selinux,pam_loginuid,pam_selinux,pam_keyinit,pam_namespace,
➥ pam_keyinit,pam_limits,pam_systemd,pam_unix,pam_gnome_keyring,pam_umask acct=
➥ "dwalsh" exe="/usr/libexec/gdm-session-worker" hostname=fedora addr=? 
➥ terminal=/dev/tty2 res=success'UID="root" AUID="dwalsh"
```

如果您使用 Podman 命令启动了容器，审计子系统会在审计日志中记录您的 UID。如果容器是通过 Docker 启动的，它将 `-1` 记录为 `loginuid`。想象一下，如果您的系统被容器攻击，您需要回溯并检查哪个用户通过 audit.log 启动了攻击您系统的容器。

让我们通过一个例子来展示这一点。首先，成为 root 用户，并使用 `auditctl` 在 /etc/passwd 文件上设置监视：

```
# auditctl -w /etc/passwd -p wa -k passwd
```

现在运行一个使用 Docker 的 `--privileged` 容器，它接触宿主的 /etc/passwd 文件：

```
# docker run --privileged -v /:/host registry.access.redhat.com/ubi8-
➥ micro:latest touch /host/etc/passwd
```

这模拟了如果 Docker 容器逃离了限制并能够修改宿主的 /etc/passwd 文件会发生什么。现在检查 audit.log，其中应该有 /etc/passwd 修改的记录。注意，审计日志显示 `auid=unset`。这就是审计日志表示修改 /etc/passwd 文件的用户 `loginuid` 的方式。如您所见，因为没有用户直接启动 Docker 守护进程，审计日志没有记录启动容器的用户：

```
# ausearch -k passwd -i
...
type=SYSCALL msg=audit(05/03/2022 08:24:52.885:464) : arch=x86_64 
➥ syscall=openat success=yes exit=3 a0=AT_FDCWD a1=0x7ffef7a9ef75 
➥ a2=O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK a3=0x1b6 items=2 ppid=6723 
➥ pid=6743 auid=unset uid=root gid=root euid=root suid=root fsuid=root 
➥ egid=root sgid=root fsgid=root tty=(none) ses=unset comm=touch 
➥ exe=/usr/bin/coreutils 
```

现在用 Podman 运行相同的命令：

```
# podman run --privileged -v /:/host registry.access.redhat.com/
➥ ubi8-micro:latest touch /host/etc/passwd
```

检查修改 /etc/passwd 文件的 Podman 容器的 audit.log，您会看到 `auid=dwalsh`。因为 Podman 遵循 fork/exec 模型，并且是由登录系统的用户启动的，该用户在 `loginuid` 中有记录，所以 audit.log 可以记录哪个用户启动了攻击系统的容器：

```
# ausearch -k passwd -i
...
type=SYSCALL msg=audit(05/03/2022 08:25:42.466:480) : arch=x86_64 
➥ syscall=openat success=no exit=EACCES(Permission denied) a0=AT_FDCWD 
➥ a1=0x7fff3d5aef59 a2=O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK a3=0x1b6 
➥ items=2 ppid=6978 pid=6986 auid=dwalsh uid=root gid=root euid=root 
➥ suid=root fsuid=root egid=root sgid=root fsgid=root tty=(none) ses=1 
➥ comm=touch exe=/usr/bin/coreutils 
➥ subj=system_u:system_r:container_t:s0:c484,c845 key=passwd
```

注意：在当前的 Fedora 中，审计子系统已被禁用。您可以通过删除 `/etc/audit/rules.d/audit.rules` 并使用 `augenrules` `--load` 命令重新生成审计规则来启用它。

这就是为什么在 2014 年，我说通过非 root 进程访问 docker.sock 比提供 root 进程或 sudo 访问更危险，因为这两种情况都会记录 `loginuid`，这意味着您可以跟踪用户在系统上的操作。当您提供对运行 docker.sock 的 root 的访问权限时，您没有任何跟踪数据。让我们在下一节中看看您如何保护内核和文件系统免受容器内进程的影响。

## 11.2 Podman 密钥处理

在运行容器时，通常需要向容器内运行的服务提供密钥。例如，这是一个需要管理员和密码来控制访问的数据库工具。另一个例子是需要一个密码才能访问另一个服务的服务。

这些应用程序的开发者不希望将密钥信息硬编码到镜像中。容器应用程序的用户必须提供密钥。您只需将密钥通过环境变量提供给应用程序即可，但这意味着如果您提交镜像，密钥也会被提交到镜像中。

Podman 提供了一个密钥机制，`podman` `secret`，允许您在不将这些密钥保存在将容器提交到镜像时添加文件或环境变量。首先，让我们看看如何创建一个密钥。

列表 11.1 在 Podman 容器中使用密钥

```
$ echo "This is my secret" > /tmp/secret               ❶
$ podman secret create my_secret /tmp/secret           ❷
b5f27b90e9b3486fb5a78d1eb
$ podman run --rm --secret my_secret ubi8 cat 
/run/secrets/my_secret                                 ❸
This is my secret
```

❶ 将您的密钥数据添加到文件中。

❷ 使用 podman secret create 命令根据文件命名一个密钥。

❸ 使用 --secret 选项将密钥泄露到容器中。

您还可以通过添加 `--secret my_secret,type=env` 标志将密钥泄露到容器中作为环境变量：

```
$ podman run --secret my_secret,type=env --name secret_ctr ubi8 bash 
➥ -c 'echo $my_secret'
This is my secret
```

如果您要将此容器提交到镜像，密钥将不会保存在镜像内部。

列表 11.2 当容器提交到镜像时，密钥不会被保存。

```
$ podman commit secret_ctr secret_img                       ❶
Getting image source signatures
Copying blob a9820c2af00a skipped: already exists 
Copying blob 3d5ecee9360e skipped: already exists 
Copying blob dc409efbefc4 done 
Copying config 501812299f done 
Writing manifest to image destination
Storing signatures
501812299f0c0cfbb032d144e6d2c2a41c5eadf229e7b76f6264ab74d9f6c069
$ podman image inspect secret_img --format 
➥ '{{ .Config.Env }}'                                      ❷
[TERM=xterm container=oci PATH=/usr/local/sbin:/usr/local/
➥ bin:/usr/sbin:/usr/bin:/sbin:/bin]
```

❶ 将 secret_ctr 提交到 secret_img 镜像中。

❷ 检查镜像以查看提交的环境变量，并注意 my_secret 环境变量没有被提交。

表 11.1 列出了所有 `podman` `secret` 命令。

表 11.1 `podman` `secret` 命令

| 命令 | 手册页 | 描述 |
| --- | --- | --- |
| `create` | `podman-secret-create(1)` | 创建一个新的密钥。 |
| `inspect` | `podman-secret-inspect(1)` | 显示一个或多个密钥的详细信息。 |
| `ls` | `podman-secret-ls(1)` | 列出所有可用的密钥。 |
| `rm` | `podman-secret-rm(1)` | 删除一个或多个密钥。 |

## 11.3 Podman 镜像信任

在许多情况下，容器镜像的用户希望指定他们信任哪些容器镜像仓库和镜像。`podman` `image` `trust` 命令允许您指定您信任的仓库。它还允许您指定要阻止的仓库。

信任注册库的位置由图像的传输和注册库主机确定。以使用此容器图像—docker://quay.io/podman/stable—为例，Docker 是传输，quay.io 是注册库主机。

注意：Podman 图像信任在远程模式下不可用，例如，在 Mac 或 Windows 箱子上。您必须在 Linux 箱子上执行此处记录的命令。如果您使用 Podman 机器，请使用 `podman` `machine` `ssh` 命令进入虚拟机。有关更多信息，请参阅附录 E 和 F。

信任策略定义在 /etc/containers/policy.json 中，它描述了信任的注册库范围（注册库和/或存储库）。信任策略可以使用公钥为签名的图像。必须以 root 用户运行 `podman` `image` `trust` 命令。

信任的范围从最具体到最不具体进行评估。换句话说，可以为整个注册库定义一个策略。或者，它可以为该注册库中的特定存储库定义策略，或者定义到注册库中特定签名的图像。在以下示例中，您拒绝从 docker.io 拉取，然后后来仅指定允许拉取 docker.io/library 图像。

以下列表包括在 policy.json 中可以使用的有效范围值，从最具体到最不具体：

```
docker.io/library/busybox:notlatest
docker.io/library/busybox
docker.io/library
docker.io
```

如果在这些范围内没有找到任何配置，则使用默认值（使用 `default` 而不是 `REGISTRY[/REPOSITORY]` 指定），如下所示。表 11.2 描述了用于注册库的有效信任值。

列表 11.3 告诉 Podman 不要从特定的容器注册库拉取图像

```
$ sudo podman image trust set -t reject docker.io                     ❶
$ podman pull alpine                                                  ❷
Trying to pull docker.io/library/alpine:latest...
Error: Source image rejected: Running image docker://alpine:latest 
➥ is rejected by policy.
$ sudo podman image trust set -t accept 
➥ docker.io/library                                                  ❸
$ podman pull alpine                                                  ❹
Trying to pull docker.io/library/alpine:latest...
Getting image source signatures
Copying blob 59bf1c3509f3 skipped: already exists 
Copying config c059bfaa84 done 
Writing manifest to image destination
Storing signatures
C059bfaa849c4d8e4aecaeb3a10c2d9b3d85f5165c66ad3a4d937758128c4d18
$ podman pull bitnami/nginx                                           ❺
Resolving "bitnami/nginx" using unqualified-search registries 
➥ (/etc/containers/registries.conf.d/999-podman-machine.conf)
Trying to pull docker.io/bitnami/nginx:latest...
Error: Source image rejected: Running image docker://bitnami/nginx:latest 
➥ is rejected by policy.
```

❶ 使用 podman 图像信任拒绝来自 docker.io 容器注册库的所有图像。

❷ 尝试从容器注册库中拉取 Alpine 图像，并查看 Podman 拒绝了该图像。

❸ 使用 Podman 图像信任设置 docker.io/library 的更具体的注册库/存储库。

❹ Podman 可以拉取 docker.io/library/alpine 图像。

❺ 从 docker.io 的其余部分拉取的图像被拒绝。

表 11.2 信任类型告诉容器引擎（如 Podman）信任哪些注册库。

| 类型 | 描述 |
| --- | --- |
| `accept` | 允许从指定的注册库拉取图像。 |
| `reject` | 不允许从指定的注册库拉取图像。 |
| `signBy` | 来自指定注册库的图像必须由指定的名称签名。 |

如果检查 policy.json 文件，您会看到由 `podman` `image` `trust` 命令添加的条目：

```
$ cat /etc/containers/policy.json
{
    "default": [
        {
            "type": "insecureAcceptAnything"
        }
    ],
    "transports": {
        "docker": {
            "docker.io": [
                {
                    "type": "reject"
                }
            ],
            "docker.io/library": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]

...   
```

您可以使用 `podman` `image` `trust` `show` 命令以更易于查看的形式显示当前设置：

```
$ podman image trust show
all          default                     accept                         
repository   docker.io                   reject                         
repository   docker.io/library           accept                      

repository   registry.access.redhat.com  signed    security@redhat.com 
https://access.redhat.com/webassets/docker/content/sigstore
repository   registry.redhat.io          signed    
➥ security@redhat.com  https://registry.redhat.io/containers/sigstore
docker-daemon                            accept   
```

通过 `accep`t 和 `reject` 标志，您可以设置信任和拒绝哪些注册库。如果您想锁定生产系统中图像的来源，您可以更改系统的 `default` 策略为 `reject` 来自任何注册库的图像。您想要允许的所有图像都必须来自特定的注册库：

```
$ sudo podman image trust set --type=reject default
$ podman image trust show
all          default                     reject                      

repository   docker.io                   reject                      

repository   docker.io/library           accept                      

repository   registry.access.redhat.com  signed    security@redhat.com 
https://access.redhat.com/webassets/docker/content/sigstore
repository   registry.redhat.io          signed    
➥ security@redhat.com  https://registry.redhat.io/containers/sigstore
docker-daemon                            accept   
```

在你的系统上设置这些设置后，Podman 接受来自 docker.io/library 的图像和来自 registry.redhat.io 的签名图像。来自其他注册表的图像都将被拒绝。Podman 还允许直接从 `docker-daemon` 拉取图像。

不要忘记恢复默认的 policy.json：

```
$ sudo cp /tmp/policy.json /etc/containers/policy.json 
```

Podman 支持使用来自容器注册表的签名图像。红帽公司签名并分发其图像。让我们看看你如何也能签名图像。

### 11.3.1 Podman 图像签名

签名图像的一种方式是使用 GNU Privacy Guard ([`gnupg.org`](https://gnupg.org)) 密钥。Podman 可以在将图像推送到远程注册表之前对其进行签名，这被称为 *简单签名*。你可以配置 Podman 和其他容器引擎，要求图像必须使用特定的签名进行签名。所有未签名的图像都将被拒绝。

首先，你需要创建一个 GPG 密钥对或选择一个预制的密钥对。你可以通过运行 `gpg` `--full-gen-key` 并遵循交互式对话框来生成新的 GPG 密钥。有关创建密钥的说明，请参阅以下网页：[`mng.bz/JV9V`](http://mng.bz/JV9V)。

以下是一个使用默认参数创建简单密钥的示例。请确保使用你自己的电子邮件地址：

```
$ gpg --batch --passphrase '' --quick-gen-key dwalsh@redhat.com default 
➥ default
```

大多数容器注册表都不理解图像签名；它们只是为容器图像提供远程存储。如果你想签名一个图像，你需要自己分发签名，通常使用网络服务器。你可以配置 Podman 和其他容器引擎从该网络服务检索签名。

在以下示例中，你将在本地机器上创建一个运行着的网络服务来演示图像签名。Podman 能够通过单个命令推送和签名图像。Podman 读取注册表配置文件 /etc/containers/registries.d/default.yaml 中的签名位置。

检查 default.yaml 文件以找到 `sigstore-staging` 标志并查看 Podman 存储签名的默认位置：

```
sigstore-staging: file:///var/lib/containers/sigstore
```

`sigstore-staging` 标志告诉 Podman 将签名存储在 /var/lib/containers/sigstore 目录中。当你想让其他用户使用这些签名来验证你的图像时，你需要将这些图像上传到网络服务器。现在你已经准备好测试简单的签名了，首先签名 ubi8 图像，然后设置 Podman 使用签名来验证拉取的图像。

签名并推送图像

在开始本节之前，你应该备份几个安全文件，以便稍后恢复：

```
$ sudo cp /etc/containers/registries.d/default.yaml 
➥ /etc/containers/policy.json /tmp 
```

让我们从注册表中拉取一个图像并添加一个签名，然后将它推回注册表。请确保使用你自己的注册表账户、图像和之前创建的 GPG 密钥：

```
$ sudo podman pull quay.io/rhatdan/myimage
Trying to pull quay.io/rhatdan/myimage:latest...
...
2c7e43d880382561ebae3fa06c7a1442d0da2912786d09ea9baaef87f73c29ae
$ podman login quay.io/rhatdan
Username: rhatdan
Password:
Login Succeeded!
$ sudo -E GNUPGHOME=$HOME/.gnupg \
    podman push --tls-verify=false --sign-by dwalsh@redhat.com 
➥ quay.io/rhatdan/myimage
...
Storing signatures
```

在 sigstore-staging 目录 /var/lib/containers/sigstore 中查找仓库名称 rhatdan。你会看到有一个新的签名可用，这是由 `podman` `push` 命令创建的。请确保使用你自己的注册表账户名：

```
$ sudo ls /var/lib/containers/sigstore/rhatdan/
'myimage@sha256=0460a9d13a806e124639b23e9d6ffa1e5773f7bef91469bee6ac88
➥ a4be213427'
```

现在你已经签了镜像，你需要设置一个 Web 服务器来提供签名，并配置 Podman 和其他容器引擎以使用签名和已签名的镜像。

配置 Podman 拉取已签名的镜像

当配置 Podman 使用签名来验证镜像时，你需要配置系统以检索签名。通常，你会在 Web 服务上共享签名。你可以通过在`/etc/containers/registries.d/default.yaml`文件中配置`sigstore`标志来识别存储签名的网站。Podman 从该网站下载这些签名。

对于这个例子，你将创建一个在本地主机`8000`端口上运行的 Web 服务。将`sigstore:` `http://localhost:8000` Web 服务器添加到默认的`default.yaml`文件中。这将告诉 Podman 在拉取镜像时从该 Web 服务器检索签名。Podman 根据镜像的名称及其摘要查找签名：

```
$ echo "  sigstore: http://localhost:8000" | sudo tee --append 
➥ /etc/containers/registries.d/default.yaml
```

对于这个例子，在本地预演签名存储`/var/lib/containers/sigstore`中使用`python3`启动一个新的服务器：

```
$ cd /var/lib/containers/sigstore && python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

在另一个窗口中，从本地存储中删除 quay.io/rhatdan/myimage，因为你想要带签名的拉取：

```
$ podman rmi quay.io/rhatdan/myimage
Untagged: quay.io/rhatdan/myimage:latest
Deleted: 2c7e43d880382561ebae3fa06c7a1442d0da2912786d09ea9baaef87f73c29ae
```

你需要为 quay.io/rhatdan 存储库设置镜像信任，并将 publickey.gpg 公钥分配给用于验证 dwalsh@redhat.com 签名的镜像：

```
$ sudo podman image trust set -f /tmp/publickey.gpg quay.io/rhatdan
```

之前的 Podman 命令将以下段落添加到`/etc/containers/policy.json`文件中：

```
...
"transports": {
    "docker": {
        "quay.io/rhatdan": [
            {
                  "type": "signedBy",
                  "keyType": "GPGKeys",
                  "keyPath": "/tmp/publickey.gpg"
            }
        ],
...
```

你还没有创建`keyPath`文件`/tmp/publickey.gpg`。使用以下 GPG 命令创建它：

```
$ gpg --output /tmp/publickey.gpg --armor --export dwalsh@redhat.com
```

现在，你可以拉取已签名的镜像：

```
$ podman pull quay.io/rhatdan/myimage
Trying to pull quay.io/rhatdan/myimage:latest...
...
Writing manifest to image destination
Storing signatures
2c7e43d880382561ebae3fa06c7a1442d0da2912786d09ea9baaef87f73c29ae
```

这成功了！尽管如此，你仍然不确定它是否使用了签名。通过尝试从没有签名的仓库中拉取另一个镜像来证明给自己，这将失败：

```
$ podman pull quay.io/rhatdan/podman
Trying to pull quay.io/rhatdan/podman:latest...
Error: Source image rejected: A signature was required, 
➥ but no signature exists
```

确保将所有设置恢复到默认值：

```
$ sudo cp /tmp/default.yaml /etc/containers/registries.d/default.yaml
$ sudo cp /tmp/policy.json /etc/containers/policy.json
```

此外，停止在另一个终端中启动的本地主机 Web 服务器。表 11.3 描述了你需要设置的必要基础设施，以允许在环境中使用简单的签名。

表 11.3 简单签名所需的基础设施

| 要求 | 描述 |
| --- | --- |
| GPG 私钥 | 你需要一个 GPG 密钥对，其中私钥用于签名镜像的服务。 |
| 签名 Web 服务器 | Web 服务器必须运行在可以访问签名存储的地方。 |

一旦你设置了使用简单签名的必要基础设施，你将需要了解每个使用和验证签名的客户端的要求。表 11.4 列出了这些要求。

表 11.4 简单签名所需客户端配置

| 要求 | 描述 |
| --- | --- |
| GPG 公钥(/tmp/publickey.gpg) | 用于签名的公钥必须在拉取已签名镜像的任何机器上存在。 |
| 客户端 sigstore 配置 | 签名 Web 服务器必须在所有需要拉取已签名镜像的系统的`/etc/containers/registries.d/*.yaml`文件中配置为 sigstore。 |
| 客户端图像信任配置 | 图像信任必须在使用这些图像的每个容器引擎系统上进行配置。 |

## 11.4 Podman 图像扫描

Podman 不是一个图像扫描器；它将这项任务留给了其他工具。但 Podman 确实有一个很好的功能，使得扫描器扫描图像变得更加容易。Podman 可以直接挂载可扫描的图像。扫描器查看图像挂载的内容，而无需执行图像中的任何代码。回想一下，您不能在没有首先进入用户命名空间的情况下以 rootless 模式挂载容器或图像。执行 `podman image mount` 命令以显示错误：

```
$ podman image mount ubi8
Error: cannot run command "podman image mount" in rootless mode, must 
➥ execute `podman unshare` first
```

在下一个示例中，您首先使用 `podman unshare` 进入用户命名空间，然后挂载 ubi8 图像。最后，更改目录到挂载目录，并运行一个 `find` 命令以定位图像中的所有 `setuid` 二进制文件。注意，您使用来自主机操作系统的工具来扫描图像：

```
$ podman unshare
# podman image mount
# mnt=$(podman image mount ubi8)
# echo $mnt
/home/dwalsh/.local/share/containers/storage/overlay/05ddfb76c5eb2146646c70
➥ e20db21a35dfec2215f130ce8bd04fce530142cfbd/merged
# cd $mnt
# /usr/bin/find . -user root -perm -4000
./usr/libexec/dbus-1/dbus-daemon-launch-helper
./usr/bin/chage
./usr/bin/mount
./usr/bin/umount
./usr/bin/newgrp
./usr/bin/gpasswd
./usr/bin/passwd
./usr/bin/su
./usr/sbin/userhelper
./usr/sbin/unix_chkpwd
./usr/sbin/pam_timestamp_check
```

使用图像内部工具扫描图像是不安全的，因为图像的攻击者可以修改扫描工具。Podman 使得扫描器更容易完成他们的工作。

### 11.5.1 只读容器

我经常谈论生产环境中的容器与开发环境中的容器。当容器化应用程序处于开发状态时，能够写入容器图像并在以后提交该图像是有用的。尽管这很常见，但大多数人会在实际构建图像时切换到使用 Containerfiles。底线是，一旦开发人员将软件移交给质量工程团队，他们期望内容被视为只读。

当在生产环境中运行容器时，我认为以只读模式运行图像是有意义的。想象一下，您正在运行一个被黑客攻击的应用程序。黑客首先想要做的是将后门写入应用程序；然后，下一次容器或应用程序启动时，容器已经有了漏洞。如果图像是只读的，黑客将无法留下后门，并被迫从头开始循环。

`--read-only` 选项阻止应用程序将内容写入图像，并强制应用程序只将内容写入 tmpfs 文件系统或添加到容器中的卷。有时您可能想要阻止容器在您的系统上的任何位置写入，并且只读取或执行容器内的代码。以只读模式运行容器的另一个好处是，您可以捕获到您不知道容器正在写入图像的错误。最后，在像 overlayfs 这样的写时复制文件系统上写入，几乎总是比写入卷或 tmpfs 慢：

```
$ podman run --read-only ubi8 touch /foo
touch: cannot touch '/foo': Read-only file system
```

以 rootless 模式运行的一个问题是，应用程序通常期望写入 /run、/tmp 和 /var/tmp。Podman 通过在这些位置自动挂载 tmpfs 文件系统来管理这个问题：

```
$ podman run --read-only ubi8 touch /run/foo
```

由于一些用户认为允许容器化应用程序在任何地方写入，即使在 tmpfs 挂载上，也太不安全了，Podman 添加了`--read-only-tmpfs`选项。`--read-only-tmpfs`选项在`--read-only`模式下运行时添加了/run、/tmp 和/var/tmp tmpfs。如果您想禁用此功能，可以使用`–-read-only-tmpfs=false`标志：

```
$ podman run --read-only-tmpfs=false --read-only ubi8 touch /run/foo
touch: cannot touch '/run/foo': Read-only file system
```

## 11.5 深度安全

在安全领域，有一个常见的想法，即*深度安全*。根据这一理念，应使用多层或工具来保护资产。这一理念的典型类比是古代城堡的安全，通常建在山顶上，有多个城墙，有护城河，甚至还有更多的安全功能。攻击者需要突破所有这些层才能到达统治者。

容器安全的工作方式与此类似。Podman 使用 Linux 提供的所有安全机制，为您提供深度安全。

### 11.5.1 Podman 同时使用所有安全机制

Podman 容器可以运行本章提到的所有安全机制。这意味着被黑客攻击的容器需要找到一种方法来逃离只读文件系统、命名空间、丢弃的能力、SELinux、seccomp 等等，以获得对您系统的访问权限。

在某些情况下，您可能需要放宽一些安全机制，以便容器可以运行。理解如何处理本章讨论的安全功能总是比仅仅使用`--privileged`标志运行容器要好，这个标志会关闭您所有的防御。

Podman 旨在为容器提供合理的安全包装，但它需要允许通用容器成功。了解您的容器应用程序的安全需求和 Podman 的安全功能，可以使您提高容器安全包装。如果您知道您的容器不需要以 root 身份运行，就不要以 root 身份启动它。如果您的容器不需要任何 Linux 能力，就丢弃它们。无 root 容器比有 root 容器更好。您还可以考虑以只读模式或在内部分离的用户命名空间中运行容器。通过简单地采用这些措施，您就有能力使您的容器化应用程序的城堡墙壁更加坚固。

### 11.5.2 您应该在何处运行您的容器？

我将留给您最后一个想法。在本章的开头，我谈到了住在不同类型庇护所中的三只猪——独立房屋、联排别墅和公寓楼——每个都比上一个稍微不安全一些。容器安全可以做得比住在单独住宅单元的猪更好，因为单元可以堆叠在一起。

假设你拥有两个不同的容器：一个用于网页前端的容器和一个包含信用卡数据的数据库容器。如果你想确保它们是隔离的，你可以在系统内部将它们放在同一个容器中，或者更好的做法是将它们放入容器中，但分别放入不同的虚拟机中，最后再将这些虚拟机放在不同的机器上。你将能够将你的网页前端放入一个运行在容器内部的虚拟机中，该虚拟机位于你的 DMZ 内部，面向互联网。你可以在将你的数据库放在你的私有网络内部的同时完成所有这些操作，而不会限制你的网页前端对网络的访问。可能性几乎是无限的。

## 摘要

+   容器安全有许多不同的方面，包括运行容器的隔离、信任镜像和注册表、扫描镜像等。

+   深度防御意味着你的容器工具利用尽可能多的安全机制。如果某个安全机制失效，其他机制可能仍然能保护你的系统。

+   容器安全全部关乎保护 Linux 内核和宿主文件系统免受恶意容器进程的侵害。

+   设置和控制你在系统上运行的容器镜像至关重要。不要允许你的用户从互联网上运行随机应用程序。
