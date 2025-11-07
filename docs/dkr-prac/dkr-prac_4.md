## 第五部分\. 生产环境中的 Docker

最后，我们准备开始考虑在生产环境中运行 Docker。在第五部分中，我们将讨论在生产环境中运行 Docker 时需要考虑的关键操作问题。

安全性是第十四章的重点。通过实际的技术，你将真正理解 Docker 带来的安全挑战以及你可能如何解决这些问题。

备份、日志记录和资源管理在第十五章中进行了讨论，我们将向您展示如何在 Docker 环境中管理这些传统的系统管理员任务。

最后，在第十六章中，我们将探讨当事情出错时你可以做什么，涵盖 Docker 可能遇到的一些常见问题区域，以及如何在生产环境中调试容器。

## 第十四章\. Docker 和安全

| |
| --- |

**本章涵盖**

+   Docker 提供的默认安全功能

+   Docker 为提高安全性所采取的措施

+   其他方面都在做什么

+   可以采取哪些其他步骤来缓解安全担忧

+   如何使用 aPaaS 管理用户 Docker 权限，可能在多租户环境中

| |
| --- |

如 Docker 在其文档中明确指出的，对 Docker API 的访问意味着对 root 权限的访问，这就是为什么 Docker 通常需要使用 sudo 运行，或者用户必须被添加到允许访问 Docker API 的用户组（可能被称为“docker”或“dockerroot”）中。

在本章中，我们将探讨 Docker 中的安全问题。

### 14.1\. Docker 访问及其含义

你可能想知道如果用户可以运行 Docker，他们能造成什么样的损害。作为一个简单的例子，以下命令（不要运行它！）将删除你主机机器上 /sbin 中的所有二进制文件（如果你移除了虚假的 `--donotrunme` 标志）：

```
docker run --donotrunme -v /sbin:/sbin busybox rm -rf /sbin
```

值得指出的是，即使你不是 root 用户，这也同样适用。

以下命令将显示主机系统安全影子密码文件的内容：

```
docker run -v /etc/shadow:/etc/shadow busybox cat /etc/shadow
```

Docker 的不安全性通常被误解，部分原因是由于对内核中命名空间好处的误解。Linux 命名空间提供了对系统其他部分的隔离，但你拥有的隔离程度由你自行决定（如前述 `docker run` 示例所示）。此外，Linux 操作系统的所有部分并不都具有命名空间的能力。设备和内核模块是两个核心 Linux 功能的例子，它们不是命名空间化的。

| |
| --- |

##### 小贴士

Linux 命名空间是为了允许进程拥有与其他进程不同的系统视图而开发的。例如，*进程命名空间*意味着容器只能看到与该容器关联的进程——在相同主机上运行的其它进程对他们来说是不可见的。*网络命名空间*意味着容器似乎有自己的网络堆栈可供使用。命名空间已经成为了 Linux 内核的多年组成部分。

| |
| --- |

此外，因为你能够通过系统调用从容器内以 root 身份与内核交互，任何内核漏洞都可能被 Docker 容器内的 root 用户利用。当然，虚拟机也有类似的攻击可能，这是通过访问虚拟机管理程序来实现的，因为虚拟机管理程序也有针对它们的安全漏洞报告。

理解这里风险的另一种方式是将运行 Docker 容器视为（从安全角度来看）与能够通过包管理器安装任何包没有区别。运行 Docker 容器时的安全需求应该与安装包时的需求相同。如果你有 Docker，你可以以 root 身份安装软件。这也是为什么有些人认为 Docker 最好被理解为一个软件打包系统的一部分原因。

| |
| --- |

##### 小贴士

正在进行通过用户命名空间来消除这种风险的工作，它将容器中的 root 映射到主机上的非特权用户。

| |
| --- |

#### 14.1.1. 你关心吗？

由于对 Docker API 的访问等同于 root 访问权限，接下来的问题是“你关心吗？”尽管这听起来可能是一条奇怪的论断，但安全就是关于信任的，如果你信任你的用户在他们操作的环境中安装软件，那么他们运行 Docker 容器时应该没有障碍。安全困难主要出现在考虑多租户环境时。因为容器内的 root 用户在关键方面与容器外的 root 用户相同，所以有大量不同的用户在系统中拥有 root 权限可能会引起担忧。

| |
| --- |

##### 小贴士

多租户环境是指许多不同的用户共享相同资源的环境。例如，两个团队可能使用两个不同的虚拟机共享同一台服务器。通过共享硬件而不是为特定应用程序配置硬件，多租户提供了成本节约。但它也可能带来与服务可靠性和安全隔离相关的一些挑战，这些挑战可能会抵消成本节约。

| |
| --- |

一些组织采取为每个用户运行 Docker 在专用虚拟机上的方法。虚拟机可以用于安全、操作或资源隔离。在虚拟机的信任边界内，用户运行 Docker 容器以获得它们带来的性能和操作优势。这是 Google Compute Engine 采取的方法，它在用户的容器和底层基础设施之间放置一个虚拟机，以增加一层安全性和一些操作优势。Google 拥有大量的计算资源，所以他们并不介意这样做带来的开销。

### 14.2. Docker 中的安全措施

Docker 维护者已经采取了各种措施来降低运行容器时的安全风险。例如，

+   现在将某些核心挂载点（如/proc 和/sys）挂载为只读。

+   默认 Linux 权限已经降低。

+   支持第三方安全系统，如 SELinux 和 AppArmor。

在本节中，我们将更深入地探讨这些问题，以及你可以采取的一些措施来降低在系统上运行容器时的风险。

| |
| --- |

**限制能力**

正如我们已经提到的，容器中的 root 用户与主机上的 root 用户相同。但并非所有 root 用户都是平等的。Linux 为你提供了在进程内为 root 用户分配更细粒度权限的能力。

这些细粒度权限被称为*能力*，它们允许你限制用户即使作为 root 用户也能造成的损害。这项技术展示了如何在运行 Docker 容器时操作这些能力，特别是如果你不完全信任其内容时。

**问题**

你希望减少容器在主机上执行破坏性操作的能力。

**解决方案**

使用`--drop-cap`标志来降低容器可用的能力。

**Unix 信任模型**

要理解“降低能力”的含义和作用，需要一点背景知识。当 Unix 系统设计时，信任模型并不复杂。你有一些受信任的管理员（root 用户）和不受信任的用户。root 用户可以做任何事情，而标准用户只能影响自己的文件。由于系统通常在大学实验室中使用且规模较小，这种模型是合理的。

随着 Unix 模型的发展以及互联网的到来，这种模型变得越来越没有意义。像 Web 服务器这样的程序需要 root 权限来在端口 80 上提供服务内容，但它们也有效地充当了在主机上运行命令的代理。为了处理这种情况，建立了标准模式，例如绑定到端口 80 并将有效用户 ID 降低到非 root 用户。执行各种角色的用户，从系统管理员到数据库管理员，再到应用支持工程师和开发者，都可能需要系统上不同资源的细粒度访问。Unix 组在一定程度上缓解了这个问题，但任何系统管理员都会告诉你，建模这些权限需求是一个非平凡的问题。

**Linux 能力**

为了支持对特权用户管理的更细粒度方法，Linux 内核工程师开发了*能力*。这是尝试将单一的 root 权限分解成可以单独授予的功能片段。你可以通过运行`man 7 capabilities`（假设你已经安装了 man 页面）来详细了解它们。

Docker 默认关闭了一些能力。这意味着即使你在容器中有 root 权限，也有一些事情你无法做到。例如，`CAP_NET_ADMIN`能力，它允许你影响主机的网络堆栈，默认是禁用的。

表 14.1 列出了 Linux 能力，简要描述了它们允许做什么，并指出了它们在 Docker 容器中是否默认允许。

##### 表 14.1\. Docker 容器中的 Linux 功能

| **功能** | **描述** | **已开启？** |
| --- | --- | --- |
| CHOWN | 对任何文件进行所有权更改 | Y |
| DAC_OVERRIDE | 覆盖读取、写入和执行检查 | Y |
| FSETID | 修改文件时不清除 suid 和 guid 位 | Y |
| FOWNER | 在保存文件时覆盖所有权检查 | Y |
| KILL | 绕过信号上的权限检查 | Y |
| MKNOD | 使用 | Y |
| NET_RAW | 使用原始套接字和数据包套接字，并绑定到端口以进行透明代理 | Y |
| SETGID | 更改进程的组所有权 | Y |
| SETUID | 更改进程的用户所有权 | Y |
| SETFCAP | 设置文件功能 | Y |
| SETPCAP | 如果文件功能不受支持，则将功能限制应用于来自和来自其他进程 | Y |
| NET_BIND_SERVICE | 将套接字绑定到小于 1024 的端口 | Y |
| SYS_CHROOT | 使用 chroot | Y |
| AUDIT_WRITE | 写入内核日志 | Y |
| AUDIT_CONTROL | 启用/禁用内核日志 | N |
| BLOCK_SUSPEND | 使用阻止系统挂起的功能 | N |
| DAC_READ_SEARCH | 在读取文件和目录时绕过文件权限检查 | N |
| IPC_LOCK | 锁定内存 | N |
| IPC_OWNER | 绕过进程间通信对象的权限 | N |
| LEASE | 在普通文件上建立租约（监视尝试打开或截断） | N |
| LINUX_IMMUTABLE | 设置 FS_APPEND_FL 和 FS_IMMUTABLE_FL i 节点标志 | N |
| MAC_ADMIN | 覆盖强制访问控制（与 Smack Linux 安全模块（SLM）相关） | N |
| MAC_OVERRIDE | 强制访问控制更改（与 SLM 相关） | N |
| NET_ADMIN | 各种与网络相关的操作，包括 IP 防火墙更改和接口配置 | N |
| NET_BROADCAST | 未使用 | N |
| SYS_ADMIN | 一系列管理功能——更多信息请参阅 man capabilities | N |
| SYS_BOOT | 重启 | N |
| SYS_MODULE | 加载/卸载内核模块 | N |
| SYS_NICE | 操作进程的优先级 | N |
| SYS_PACCT | 打开或截断尝试的监视（进程会计） | N |
| SYS_PTRACE | 跟踪进程的系统调用和其他进程操作能力 | N |
| SYS_RAWIO | 在系统的各个核心部分执行 I/O，例如内存和 SCSI 设备命令 | N |
| SYS_RESOURCE | 控制和覆盖各种资源限制 | N |
| SYS_TIME | 设置系统时钟 | N |
| SYS_TTY_CONFIG | 在虚拟终端上进行特权操作 | N |
|  |

##### 注意

如果你没有使用 Docker 的默认容器引擎（libcontainer），这些功能在你的安装中可能不同。如果你有系统管理员并且想确保，请向他们询问。


不幸的是，内核维护者只在该系统中分配了 32 个功能，因此随着越来越多的细粒度 root 权限从内核中划分出来，功能范围已经扩大。最值得注意的是，名为`CAP_SYS_ADMIN`的功能涵盖了从更改主机的域名到超出系统范围内打开文件数量限制的各种操作。

一种极端的方法是移除容器中默认启用的所有 Docker 功能，并查看哪些停止工作。在这里，我们启动一个 bash shell，移除了默认启用的功能：

```
$ docker run -ti --cap-drop=CHOWN --cap-drop=DAC_OVERRIDE \
--cap-drop=FSETID --cap-drop=FOWNER --cap-drop=KILL --cap-drop=MKNOD \
--cap-drop=NET_RAW --cap-drop=SETGID --cap-drop=SETUID \
--cap-drop=SETFCAP --cap-drop=SETPCAP --cap-drop=NET_BIND_SERVICE \
--cap-drop=SYS_CHROOT --cap-drop=AUDIT_WRITE debian /bin/bash
```

如果你从这个 shell 运行你的应用程序，你可以看到它在哪里无法按预期工作，并重新添加所需的功能。例如，你可能需要更改文件所有权的功能，因此你需要取消上一段代码中`FOWNER`功能的删除才能运行你的应用程序：

```
$ docker run -ti --cap-drop=CHOWN --cap-drop=DAC_OVERRIDE \
--cap-drop=FSETID  --cap-drop=KILL --cap-drop=MKNOD \
--cap-drop=NET_RAW --cap-drop=SETGID --cap-drop=SETUID \
--cap-drop=SETFCAP --cap-drop=SETPCAP --cap-drop=NET_BIND_SERVICE \
--cap-drop=SYS_CHROOT --cap-drop=AUDIT_WRITE debian /bin/bash
```


##### 小贴士

如果你想要启用或禁用所有功能，可以使用`all`而不是特定的功能，例如`docker run -ti --cap-drop=all ubuntu bash`。


**讨论**

如果你使用 bash shell 运行一些基本命令，并且禁用了所有功能，你会发现它相当可用。然而，当运行更复杂的应用程序时，你的体验可能会有所不同。


##### 警告

值得明确的是，许多这些功能与影响系统上其他用户对象的 root 功能相关，而不是 root 自己的对象。例如，如果 root 用户在容器中，并且通过卷挂载访问主机的文件，他们仍然可以更改主机上 root 的文件的所有权。因此，仍然值得确保应用程序尽快降级为非 root 用户，以保护系统，即使所有这些功能都已关闭。


这种微调容器功能的能力意味着使用`--privileged`标志来运行`docker run`应该是多余的。需要功能的过程将是可审计的，并且受主机管理员控制。

|  |

**一个“不良”的 Docker 镜像用于扫描**

在 Docker 生态系统中，一个迅速认识到的问题是漏洞——如果你有一个不变的镜像，你也不会得到任何安全修复。如果你遵循 Docker 最佳实践中的镜像最小化，这可能不是问题，但很难判断。

图像扫描器是为了解决这个问题而创建的——一种识别镜像问题的方法，但这仍然留下了如何评估它们的问题。

**问题**

你想要确定图像扫描仪的有效性。

**解决方案**

创建一个“已知不良”的镜像来测试你的扫描器。

我们在工作中遇到了这个问题。存在许多 Docker 镜像扫描器（如 Clair），但商业产品声称可以更深入地检查镜像，以确定其中可能存在的任何潜在问题。

但没有包含已知和记录的漏洞的图像，我们可以用它来测试这些扫描器的有效性。这几乎不出所料，因为大多数图像不会宣传它们自己的不安全性！

因此，我们发明了一个已知不良的图像。该图像可供下载：

```
$ docker pull imiell/bad-dockerfile
```

原则很简单：创建一个 Dockerfile 来构建一个充满记录漏洞的图像，并将该图像指向您的候选扫描器。

Dockerfile 的最新版本可在[`github.com/ianmiell/bad-dock`](https://github.com/ianmiell/bad-dock)找到。它仍在变化中，所以这里没有打印出来。然而，它的形式非常简单：

```
FROM <base image>                           *1*
 RUN <install 'bad' software>               *2*
 COPY <copy 'bad' software in>              *2*
 [...]
CMD echo 'Vulnerable image' && /bin/false   *3*
```

+   ***1*** **参考 bad-dockerfile 存储库使用 centos 图像，但您可能希望将其替换为与您的基图像更接近的一个。**

+   ***2*** **各种 RUN/COPY/ADD 命令会将已知漏洞的软件安装到图像中。**

+   ***3*** **图像的 CMD 指令出于明显的原因，尽可能避免自己被运行。**

图像包含一系列漏洞，旨在将扫描器推到极限。

在最简单的情况下，图像使用包管理器安装已知漏洞的软件。在每个类别中，Docker 图像试图包含不同严重程度的漏洞。

通过（例如）`COPY`受漏洞影响的 JavaScript、使用特定语言的包管理器（例如 npm 用于 JavaScript、gem 用于 Ruby 和 pip 用于 Python）安装受漏洞影响的代码，甚至编译特定版本的 bash（一个带有臭名昭著的 Shellshock 漏洞的版本）并将其放置在意外位置以避免许多扫描技术，进行更复杂的漏洞放置。

**讨论**

您可能认为最好的扫描解决方案是能够捕获最多 CVE 的解决方案。但这并不一定是事实。显然，如果扫描器能够发现图像中存在漏洞，这是很好的。然而，除了这一点之外，漏洞扫描可能更多地成为一门艺术而不是一门科学。

| |
| --- |

##### **提示**

常见漏洞披露（CVE）是一般可用软件中发现的具体漏洞的标识符。CVE 的一个例子可能是 CVE-2001-0067，其中前四位数字是发现年份，后四位是该年的已识别漏洞数量。

| |
| --- |

例如，一个漏洞可能非常严重（例如在您的宿主服务器上获得 root 权限），但非常难以利用（例如需要国家层面的资源）。您（或您负责的组织）可能对此比一个不那么严重但容易利用的漏洞更不担心。例如，如果您的系统遭到 DoS 攻击，没有数据泄露或渗透的风险，但您可能会因此失去业务，所以您会更关注修补这个问题，而不是需要数万美元计算能力的某些晦涩的加密攻击。

| |
| --- |

##### 什么是 DoS 攻击？

DoS 代表“服务拒绝”。这意味着一种攻击，会导致你的系统应对需求的能力降低。服务拒绝攻击可能会使你的 Web 服务器超负荷，以至于无法响应用户的合法请求。

| |
| --- |

还值得考虑的是，漏洞实际上是否存在于正在运行的容器中。Apache Web 服务器的旧版本可能存在于镜像中，但如果容器从未实际运行过，那么漏洞实际上是可以忽略的。这种情况经常发生。包管理器通常会引入一些不必要的依赖项，仅仅是因为它使依赖项的管理变得简单。

如果安全性是一个很大的担忧，那么拥有小图像（参见第七章）可以成为另一个原因——即使某个软件未被使用，它仍然可能出现在安全扫描中，浪费你的组织在确定是否需要打补丁时的时间。

希望这项技术在你考虑哪种扫描器适合你时能给你一些思考。一如既往，这是成本、你需要什么以及你愿意为获得正确解决方案付出多少努力之间的平衡。

| |
| --- |

### 14.3\. 保护 Docker 的访问

防止 Docker 守护进程的不安全使用最好的方法是完全防止任何使用。

你可能第一次遇到受限访问是在安装 Docker 并需要使用`sudo`运行 Docker 本身时。技术 41 描述了如何选择性地允许本地机器上的用户使用 Docker 而不受此限制。

但如果你有用户从另一台机器连接到 Docker 守护进程，这并没有帮助。我们将探讨在那些情况下提供更多安全性的几种方法。

| |
| --- |

**Docker 实例上的 HTTP 身份验证**

在技术 1 中，你看到了如何打开对守护进程的网络访问，而在技术 4 中，你看到了如何使用 socat 窃听 Docker API。

这种技术结合了这两者：你将能够远程访问你的守护进程并查看响应。访问权限仅限于具有用户名/密码组合的用户，因此它稍微安全一些。作为额外的好处，你不需要重新启动 Docker 守护进程就能实现它——启动一个容器守护进程。

**问题**

你希望你的 Docker 守护进程上提供基本身份验证和网络访问。

**解决方案**

使用 HTTP 身份验证暂时与他人共享你的 Docker 守护进程。

图 14.1 展示了该技术的最终架构。

##### 图 14.1\. 基本身份验证的 Docker 守护进程架构

![](img/14fig01_alt.jpg)

| |
| --- |

##### **注意**

本讨论假设你的 Docker 守护进程正在使用 Docker 默认的 Unix 套接字访问方法，位于/var/run/docker.sock。

| |
| --- |

本技术中的代码可在[`github.com/docker-in-practice/docker-authenticate`](https://github.com/docker-in-practice/docker-authenticate)找到。以下列出的是该存储库中 Dockerfile 的内容，用于创建本技术的镜像。

##### 列表 14.1\. Dockerfile

```
FROM debian
RUN apt-get update && apt-get install -y \
nginx apache2-utils                                           *1*
 RUN htpasswd -c /etc/nginx/.htpasswd username                *2*
 RUN htpasswd -b /etc/nginx/.htpasswd username password       *3*
 RUN sed -i 's/user .*;/user root;/' \
/etc/nginx/nginx.conf                                         *4*
 ADD etc/nginx/sites-enabled/docker \
/etc/nginx/sites-enabled/docker                               *5*
 CMD service nginx start && sleep infinity                    *6*
```

+   ***1*** **确保所需的软件已更新并安装**

+   ***2*** **为名为 username 的用户创建密码文件**

+   ***3*** **将名为 username 的用户的密码设置为“password”**

+   ***4*** **Nginx 需要以 root 用户身份运行以访问 Docker Unix 套接字，因此您需要将用户行替换为“root”用户详情。**

+   ***5*** **复制 Docker 的 nginx 站点文件（列表 14.8）**

+   ***6*** **默认情况下，启动 nginx 服务并无限期等待**

使用`htpasswd`命令设置的`.htpasswd`文件包含在允许（或拒绝）访问 Docker 套接字之前需要检查的凭据。如果您自己构建此镜像，您可能希望在这两个步骤中更改`username`和`password`以自定义访问 Docker 套接字的凭据。

| |
| --- |

##### 警告

请务必不要分享这张图片，因为它将包含您设置的密码！

| |
| --- |

以下列出的是 Docker 的 nginx 站点文件。

##### 列表 14.2\. /etc/nginx/sites-enabled/docker

```
upstream docker {
  server unix:/var/run/docker.sock;                   *1*
 }
server {
  listen 2375 default_server;                         *2*
   location / {
    proxy_pass http://docker;                         *3*
     auth_basic_user_file /etc/nginx/.htpasswd;       *4*
     auth_basic "Access restricted";                  *5*
   }
}
```

+   ***1*** **在 nginx 中将 docker 位置定义为指向 Docker 的域套接字**

+   ***2*** **监听端口 2375（Docker 的标准端口）**

+   ***3*** **将这些请求代理到之前定义的 docker 位置**

+   ***4*** **定义要使用的密码文件**

+   ***5*** **通过密码限制访问**

现在以守护进程容器的形式运行镜像，映射主机机器所需资源：

```
$ docker run -d --name docker-authenticate -p 2375:2375 \
  -v /var/run:/var/run dockerinpractice/docker-authenticate
```

这将以`docker-authenticate`的名称在后台运行容器，以便您可以稍后引用。容器的主机端口 2375 被暴露，并且容器通过挂载包含 Docker 套接字的默认目录作为卷来获得对 Docker 守护进程的访问权限。如果您使用的是带有您自己的 username 和 password 的自定义构建镜像，您需要在此处将镜像名称替换为您自己的。

网络服务现在将启动并运行。如果您使用您设置的 username 和 password `curl`该服务，您应该看到 API 响应：

```
$ curl http://username:password@localhost:2375/info                      *1*
 {"Containers":115,"Debug":0, >                                          *2*
 "DockerRootDir":"/var/lib/docker","Driver":"aufs", >
"DriverStatus":[["Root Dir","/var/lib/docker/aufs"], >
["Backing Filesystem","extfs"],["Dirs","1033"]], >
"ExecutionDriver":"native-0.2", >
"ID":"QSCJ:NLPA:CRS7:WCOI:K23J:6Y2V:G35M:BF55:OA2W:MV3E:RG47:DG23", >
"IPv4Forwarding":1,"Images":792, >
"IndexServerAddress":"https://index.docker.io/v1/", >
"InitPath":"/usr/bin/docker","InitSha1":"", >
"KernelVersion":"3.13.0-45-generic", >
"Labels":null,"MemTotal":5939630080,"MemoryLimit":1, >
"NCPU":4,"NEventsListener":0,"NFd":31,"NGoroutines":30, >
"Name":"rothko","OperatingSystem":"Ubuntu 14.04.2 LTS", >
"RegistryConfig":{"IndexConfigs":{"docker.io": >
{"Mirrors":null,"Name":"docker.io", >
"Official":true,"Secure":true}}, >
"InsecureRegistryCIDRs":["127.0.0.0/8"]},"SwapLimit":0}
```

+   ***1*** **将 username:password 放入 curl 的 URL 中，以及@符号后面的地址。此请求是 Docker 守护进程 API 的/info 端点。**

+   ***2*** **Docker 守护进程的 JSON 响应**

完成后，使用以下命令删除容器：

```
$ docker rm -f docker-authenticate
```

现在访问已被撤销。

**使用 docker 命令？**

读者可能会想知道其他用户是否能够使用`docker`命令连接——例如，如下所示：

```
docker -H tcp://username:password@localhost:2375 ps
```

在撰写本文时，认证功能并未内置到 Docker 本身。但我们创建了一个可以处理认证并允许 Docker 连接到守护进程的镜像。只需按以下方式使用镜像：

```
$ docker run -d --name docker-authenticate-client \      *1*
   -p 127.0.0.1:12375:12375 \                            *2*
   dockerinpractice/docker-authenticate-client \         *3*
   192.168.1.74:2375 username:password                   *4*
```

+   ***1*** **在后台运行客户端容器并给它命名**

+   ***2*** **公开一个端口以连接 Docker 守护进程，但仅限于来自本地机的连接**

+   ***3*** **我们制作的允许与 Docker 进行认证连接的镜像**

+   ***4*** **图像的两个参数：指定认证连接另一端的位置，以及用户名和密码（这两个都应该根据您的设置适当替换）**

注意，`localhost`或`127.0.0.1`不适用于指定认证连接的另一端——如果您想在单个主机上尝试，您必须使用`ip addr`来识别您的机器的外部 IP 地址。

您现在可以使用以下命令使用认证连接：

```
docker -H localhost:12375 ps
```

请注意，由于某些实现限制，交互式 Docker 命令（带有`-i`参数的`run`和`exec`）无法通过此连接工作。

**讨论**

在这项技术中，我们向您展示了如何在受信任的网络中为您的 Docker 服务器设置基本认证。在下一项技术中，我们将探讨加密流量，这样窃听者就不能窥探您在做什么，甚至注入恶意数据或代码。


##### 警告

这种技术为您提供了基本的*认证*，但它并不提供严重的*安全性*（特别是，能够监听您网络流量的人可以拦截您的用户名和密码）。设置使用 TLS 加密的服务器要复杂得多，将在下一技术中介绍。

|  |
|  |

**保护您的 Docker API**

在这项技术中，我们将展示如何通过 TCP 端口将您的 Docker 服务器向他人开放，同时确保只有受信任的客户端可以连接。这是通过创建一个只有受信任的主机才会获得的秘密密钥来实现的。只要这个受信任的密钥在服务器和客户端机器之间保持秘密，Docker 服务器应该保持安全。

**问题**

您希望您的 Docker API 通过端口安全地提供服务。

**解决方案**

创建一个自签名证书，并使用`--tls-verify`标志运行 Docker 守护进程。

这种安全方法依赖于在服务器上创建所谓的*密钥文件*。这些文件是通过特殊工具创建的，确保在没有*服务器密钥*的情况下难以复制。图 14.2 概述了这是如何工作的。

##### 图 14.2\. 关键设置和分发

![](img/14fig02_alt.jpg)


##### 小贴士

*服务器密钥*是一个文件，它包含一个只有服务器知道的秘密数字，并且需要读取使用服务器（所谓的*客户端密钥*）提供的秘密密钥文件加密的消息。一旦密钥被创建并分发，它们就可以用来确保客户端和服务器之间的连接安全。


**设置 Docker 服务器证书**

首先，您创建证书和密钥。

生成密钥需要 OpenSSL 包，您可以通过在终端中运行`openssl`来检查它是否已安装。如果没有安装，您需要先安装它，然后才能使用以下代码生成证书和密钥。

##### 列表 14.3\. 使用 OpenSSL 创建证书和密钥

```
$ sudo su                                                                  *1*
 $ read -s PASSWORD                                                        *2*
 $ read SERVER                                                             *2*
 $ mkdir -p /etc/docker                                                    *3*
 $ cd /etc/docker                                                          *3*
 $ openssl genrsa -aes256 -passout pass:$PASSWORD \
-out ca-key.pem 2048                                                       *4*
 $ openssl req -new -x509 -days 365 -key ca-key.pem -passin pass:$PASSWORD \
-sha256 -out ca.pem -subj "/C=NL/ST=./L=./O=./CN=$SERVER"                  *5*
 $ openssl genrsa -out server-key.pem 2048                                 *6*
 $ openssl req -subj "/CN=$SERVER" -new -key server-key.pem \
-out server.csr                                                            *7*
 $ openssl x509 -req -days 365 -in server.csr -CA ca.pem -CAkey ca-key.pem
-passin "pass:$PASSWORD" -CAcreateserial \
-out server-cert.pem                                                       *8*
 $ openssl genrsa -out key.pem 2048                                        *9*
 $ openssl req -subj '/CN=client' -new -key key.pem\
-out client.csr                                                            *10*
 $ sh -c 'echo "extendedKeyUsage = clientAuth" > extfile.cnf'
$ openssl x509 -req -days 365 -in client.csr -CA ca.pem -CAkey ca-key.pem \
-passin "pass:$PASSWORD" -CAcreateserial -out cert.pem \
-extfile extfile.cnf                                                       *11*
 $ chmod 0400 ca-key.pem key.pem server-key.pem                            *12*
 $ chmod 0444 ca.pem server-cert.pem cert.pem                              *13*
 $ rm client.csr server.csr                                                *14*
```

+   ***1*** **确保您是 root 用户。**

+   ***2*** **输入您的证书密码和您将用于连接 Docker 服务器的服务器名。**

+   ***3*** **如果不存在，则创建 docker 配置目录，并进入该目录。**

+   ***4*** **使用 2048 位安全性生成证书颁发机构(CA) .pem 文件。**

+   ***5*** **使用您的密码和地址为 CA 密钥签名，有效期为一年。**

+   ***6*** **生成一个 2048 位安全性的服务器密钥。**

+   ***7*** **使用您的主机名处理服务器密钥。**

+   ***8*** **使用您的密码为密钥签名，有效期为一年。**

+   ***9*** **生成一个 2048 位安全性的客户端密钥。**

+   ***10*** **将密钥作为客户端密钥处理。**

+   ***11*** **使用您的密码为密钥签名，有效期为一年。**

+   ***12*** **将服务器文件的权限更改为 root 只读。**

+   ***13*** **将客户端文件的权限更改为所有人只读。**

+   ***14*** **删除遗留文件。**


##### **提示**

可能有一个名为 CA.pl 的脚本已安装在您的系统上，它可以简化此过程。在这里，我们展示了原始的`openssl`命令，因为它们更具指导性。


**设置 Docker 服务器**

接下来，您需要在您的 Docker 守护进程配置文件中设置 Docker opts，以指定用于加密通信的密钥（有关如何配置和重启 Docker 守护进程的说明，请参阅附录 B）。

##### 列表 14.4\. 使用新密钥和证书的 Docker 选项

```
DOCKER_OPTS="$DOCKER_OPTS --tlsverify"                 *1*
 DOCKER_OPTS="$DOCKER_OPTS \
--tlscacert=/etc/docker/ca.pem"                        *2*
 DOCKER_OPTS="$DOCKER_OPTS \
--tlscert=/etc/docker/server-cert.pem"                 *3*
 DOCKER_OPTS="$DOCKER_OPTS \
--tlskey=/etc/docker/server-key.pem"                   *4*
 DOCKER_OPTS="$DOCKER_OPTS -H tcp://0.0.0.0:2376"      *5*
 DOCKER_OPTS="$DOCKER_OPTS \
-H unix:///var/run/docker.sock"                        *6*
```

+   ***1*** **告诉 Docker 守护进程您想使用 TLS 安全来保护与其的连接**

+   ***2*** **指定 Docker 服务器的 CA 文件**

+   ***3*** **指定服务器的证书**

+   ***4*** **指定服务器使用的私钥**

+   ***5*** **通过 TCP 端口 2376 将 Docker 守护进程对外部客户端开放。**

+   ***6*** **以正常方式通过 Unix 套接字在本地打开 Docker 守护进程。**

**分发客户端密钥**

接下来，您需要将密钥发送到客户端主机，以便它可以连接到服务器并交换信息。您不希望将您的秘密密钥透露给任何人，因此这些密钥需要安全地传递给客户端。一种相对安全的方法是从服务器直接使用 SCP（安全复制）将它们复制到客户端。SCP 实用程序使用与我们在下面展示的相同技术来确保数据传输的安全性，只是使用了已经设置好的不同密钥。

在客户端主机上，创建 Docker 配置文件夹在`/etc`中，就像您之前做的那样：

```
user@client:~$ sudo su
root@client:~$ mkdir -p /etc/docker
```

然后，从服务器将文件 SCP 复制到客户端。确保在以下命令中将“客户端”替换为您的客户端机器的主机名。还要确保所有文件都可以由将在客户端上运行`docker`命令的用户读取。

```
user@server:~$ sudo su
root@server:~$ scp /etc/docker/ca.pem client:/etc/docker
root@server:~$ scp /etc/docker/cert.pem client:/etc/docker
root@server:~$ scp /etc/docker/key.pem client:/etc/docker
```

**测试**

要测试您的设置，首先尝试在没有凭证的情况下向 Docker 服务器发出请求。您应该被拒绝：

```
root@client~: docker -H myserver.localdomain:2376 info
FATA[0000] Get http://myserver.localdomain:2376/v1.17/info: malformed HTTP >
response "\x15\x03\x01\x00\x02\x02". Are you trying to connect to a >
TLS-enabled daemon without TLS?
```

然后，使用凭证连接，应该返回有用的输出：

```
root@client~: docker --tlsverify --tlscacert=/etc/docker/ca.pem \
--tlscert=/etc/docker/cert.pem --tlskey=/etc/docker/key.pem \
-H myserver.localdomain:2376 info
243 info
Containers: 3
Images: 86
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 92
Execution Driver: native-0.2
Kernel Version: 3.16.0-34-generic
Operating System: Ubuntu 14.04.2 LTS
CPUs: 4
Total Memory: 11.44 GiB
Name: rothko
ID: 4YQA:KK65:FXON:YVLT:BVVH:Y3KC:UATJ:I4GK:S3E2:UTA6:R43U:DX5T
WARNING: No swap limit support
```

**讨论**

这种技术为您提供了两全其美的解决方案——一个对其他人开放的 Docker 守护进程，以及一个仅对受信任用户可访问的守护进程。确保您保管好这些密钥！

密钥管理是大多数大型组织 IT 管理流程中的一个关键方面。这肯定是一个成本，因此在实施 Docker 平台时，它可能会变得非常突出。安全地将密钥部署到容器中是一个挑战，这在大多数 Docker 平台设计中可能需要考虑。


### 14.4. Docker 外部的安全性

在您的宿主机上，安全性并不随着`docker`命令的结束而停止。在本节中，您将看到一些其他方法来保护您的 Docker 容器，这次是从 Docker 外部进行。

我们将首先介绍一些技术，这些技术可以修改您的图像，以减少外部攻击的表面积，一旦它们启动并运行。接下来的两种技术考虑了以受限方式运行容器的方法。

在这些后两种技术中，第一种演示了应用平台即服务（aPaaS）方法，这确保 Docker 在管理员设置和控制下的紧身衣中运行。作为一个例子，我们将使用 Docker 命令运行一个 OpenShift Origin 服务器（一种以管理方式部署 Docker 容器的 aPaaS）。您将看到最终用户的权限可以被管理员限制和管理，并且可以移除对 Docker 运行时的访问。

第二种方法超越了这一级别的安全性，进一步限制运行容器内可用的自由度，使用 SELinux，这是一种提供细粒度控制谁可以做什么的安全技术。


##### **提示**

SELinux 是由美国国家安全局（NSA）构建并开源的一个工具，它满足了他们对强大访问控制的需求。它已经是一段时间以来的安全标准，并且非常强大。不幸的是，当许多人遇到问题时，他们只是简单地将其关闭，而不是花时间去理解它。我们希望这里展示的技术能帮助使这种方法不那么诱人。

| |
| --- |
| |

**使用 DockerSlim 减小容器的攻击面**

在第 7.3 节中，我们讨论了几种创建小镜像的不同方法，以应对对在网络中移动的数据量的合理担忧。但还有另一个原因要做这件事——如果你的镜像内容更少，攻击者可以利用的东西也就更少。作为一个具体的例子，如果没有安装 shell，就无法在容器中获得 shell。

为你的容器建立一个“预期行为”配置文件，并在运行时强制执行，这意味着意外行为有被检测和阻止的合理机会。

**问题**

你希望将镜像减小到最基本，以减少其攻击面。

**解决方案**

使用 DockerSlim 工具分析你的镜像并修改它以减小攻击面。

这个工具旨在将 Docker 镜像减小到最基本。它可在[`github.com/docker-slim/docker-slim`](https://github.com/docker-slim/docker-slim)找到。

DockerSlim 至少以两种不同的方式减小你的 Docker 镜像。首先，它将你的镜像减小到仅包含所需文件，并将这些文件放置在单个层中。最终结果是，这个镜像比其原始的胖镜像小得多。

其次，它为你提供了一个 seccomp 配置文件。这是通过对你运行镜像的动态分析实现的。用通俗易懂的话说，这意味着它会运行你的镜像并跟踪使用哪些文件和系统调用。当 DockerSlim 分析你的运行容器时，你需要像所有典型用户一样使用该应用程序，以确保必要的文件和系统调用被捕获。

| |
| --- |

##### 警告

如果你使用这种动态分析工具来减小你的镜像，请务必确保你在分析阶段已经充分测试。这个指南使用了一个简单的镜像，但你可能有一个更复杂的镜像，更难以完全分析。

| |
| --- |

这种技术将使用一个简单的 Web 示例应用程序来展示技术。你将

+   设置 DockerSlim

+   构建镜像

+   使用 DockerSlim 工具将镜像作为容器运行

+   打击应用程序的端点

+   使用创建的 seccomp 配置文件运行精简后的镜像

| |
| --- |

##### 注意

系统调用配置文件（seccomp profile）本质上是一个白名单，列出了容器可以从其中调用的系统调用。当运行容器时，你可以根据应用程序的需求，指定具有降低或提升权限的 seccomp 配置文件。默认的 seccomp 配置文件禁用了 300 多个系统调用中的大约 45 个。大多数应用程序需要的系统调用远少于这个数量。


**设置 DockerSlim**

运行以下命令以下载并设置 docker-slim 二进制文件。

##### 列表 14.5. 下载 docker-slim 并将其安装到目录中

```
$ mkdir -p docker-slim/bin && cd docker-slim/bin                           *1*
 $ wget https://github.com/docker-slim/docker-slim/releases/download/1.18
/dist_linux.zip                                                         *2*
 $ unzip dist_linux.zip                                                    *3*
 $ cd ..                                                                   *4*
```

+   ***1*** **创建 docker-slim 文件夹和 bin 子文件夹**

+   ***2*** **从其发布文件夹获取 docker-slim zip 文件**

+   ***3*** **解压获取的 zip 文件**

+   ***4*** **移动到父目录，docker-slim**


##### 注意

这种技术已在先前的 docker-slim 版本上进行了测试。你可能想访问 GitHub 上的[`github.com/docker-slim/docker-slim/`](https://github.com/docker-slim/docker-slim/) releases，看看是否有任何更新。这不是一个快速发展的项目，所以更新可能不会太重要。


现在你已经将 docker-slim 二进制文件放在了一个 bin 子文件夹中。

**构建胖镜像**

接下来，你将构建一个使用 NodeJS 的示例应用程序。这是一个简单的应用程序，它简单地在一个端口 8000 上提供 JSON 字符串。以下命令克隆了 docker-slim 仓库，移动到示例应用程序代码，并将其 Dockerfile 构建成一个名为 sample-node-app 的镜像。

##### 列表 14.6. 构建示例 docker-slim 应用程序

```
$ git clone https://github.com/docker-slim/docker-slim.git    *1*
 $ cd docker-slim && git checkout 1.18                        *2*
 $ cd sample/apps/node                                        *3*
 $ docker build -t sample-node-app .                          *4*
 $ cd -                                                       *5*
```

+   ***1*** **克隆 docker-slim 仓库，其中包含示例应用程序**

+   ***2*** **检出 docker-slim 仓库的一个已知工作版本**

+   ***3*** **移动到 NodeJS 示例应用程序文件夹**

+   ***4*** **构建镜像，将其命名为 sample-node-app**

+   ***5*** **返回到包含 docker-slim 二进制文件的上一目录**

**运行胖镜像**

现在你已经创建了胖镜像，下一步是将它作为带有 docker-slim 包装器的容器运行。一旦应用程序初始化，你就可以通过访问应用程序端点来测试其代码。最后，将后台的 docker-slim 应用程序带到前台并等待其终止。

```
$ ./docker-slim build --http-probe sample-node-app &                        *1*
 $ sleep 10 && curl localhost:32770                                         *2*
 {"status":"success","info":"yes!!!","service":"node"}                      *3*
 $ fg                                                                       *4*
 ./docker-slim build --http-probe sample-node-app                           *5*
 INFO[0014] docker-slim: HTTP probe started...                              *5*
 INFO[0014] docker-slim: http probe - GET http://127.0.0.1:32770/ => 200    *5*
 INFO[0014] docker-slim: HTTP probe done.                                   *5*
 INFO[0015] docker-slim: shutting down 'fat' container...                   *5*
 INFO[0015] docker-slim: processing instrumented 'fat' container info...    *5*
 INFO[0015] docker-slim: generating AppArmor profile...                     *5*
 INFO[0015] docker-slim: building 'slim' image...                           *5*

 Step 1 : FROM scratch                                                      *6*
  --->                                                                      *6*
 Step 2 : COPY files /                                                      *6*
  ---> 0953a87c8e4f                                                         *6*
 Removing intermediate container 51e4e625017e                               *6*
 Step 3 : WORKDIR /opt/my/service                                           *6*
  ---> Running in a2851dce6df7                                              *6*
  ---> 2d82f368c130                                                         *6*
 Removing intermediate container a2851dce6df7                               *6*
 Step 4 : ENV PATH "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:*6*
 /bin"
                                                                            *6*
  ---> Running in ae1d211f118e                                              *6*
  ---> 4ef6d57d3230                                                         *6*
 Removing intermediate container ae1d211f118e                               *6*
 Step 5 : EXPOSE 8000/tcp                                                   *6*
  ---> Running in 36e2ced2a1b6                                              *6*
  ---> 2616067ec78d                                                         *6*
 Removing intermediate container 36e2ced2a1b6                               *6*
 Step 6 : ENTRYPOINT node /opt/my/service/server.js                         *6*
  ---> Running in 16a35fd2fb1c                                              *6*
  ---> 7451554aa807                                                         *6*
 Removing intermediate container 16a35fd2fb1c                               *6*
 Successfully built 7451554aa807                                            *6*
 INFO[0016] docker-slim: created new image: sample-node-app.slim            *6*
 $                                                                          *7*
 $
```

+   ***1*** **运行 docker-slim 二进制文件针对 sample-node-app 镜像。将进程置于后台。http-probe 将在所有公开的端口上调用应用程序**

+   ***2*** **暂停 10 秒以允许 sample-node-app 进程启动，然后点击应用程序运行的端口**

+   ***3*** **将应用程序的 JSON 响应发送到终端**

+   ***4*** **将 docker-slim 进程置于前台并等待其完成**

+   ***5*** **docker-slim 的输出第一部分显示了其工作日志。**

+   ***6*** **Docker-slim 构建“瘦”容器。**

+   ***7*** **完成时，你可能需要按 Return 键以获取提示。**

在这种情况下，“锻炼代码”只是涉及点击一个 URL 并获取响应。更复杂的应用程序将需要更多样化和多样化的探测和检查，以确保它们已被完全锻炼。

注意，根据文档，我们不需要自己击打端口 32770 的应用程序，因为我们使用了 http-probe 参数。如果您启用 HTTP 探针，它将默认在所有暴露的端口上运行 HTTP 和 HTTPS GET 请求到根 URL（“/”）。我们手动进行 `curl` 操作只是为了演示目的。

到目前为止，您已经创建了 sample-node-app.slim 版本的镜像。如果您检查 `docker images` 的输出，您会看到其大小已经大幅减少。

```
$ docker images
REPOSITORY             TAG      IMAGE ID       CREATED             SIZE
sample-node-app.slim   latest   7451554aa807   About an hour ago   14.02 MB  *1*
 sample-node-app        latest   78776db92c2a   About an hour ago   418.5 MB *2*
```

+   ***1*** **sample-node-app.slim 镜像的大小仅为 14 MB 左右。**

+   ***2*** **原始的 sample-node-app 镜像大小超过 400 MB。**

如果您比较胖样本应用程序的 `docker history` 输出与其精简版本，您会发现它们的结构相当不同。

```
$ docker history sample-node-app                                              *1*
 IMAGE         CREATED       CREATED BY                              SIZE     *2*
 78776db92c2a  42 hours ago  /bin/sh -c #(nop)  ENTRYPOINT ["node"   0 B      *2*
 0f044b6540cd  42 hours ago  /bin/sh -c #(nop)  EXPOSE 8000/tcp      0 B      *2*
 555cf79f13e8  42 hours ago  /bin/sh -c npm install                  14.71 MB *2*
 6c62e6b40d47  42 hours ago  /bin/sh -c #(nop)  WORKDIR /opt/my/ser  0 B      *2*
 7871fb6df03b  42 hours ago  /bin/sh -c #(nop) COPY dir:298f558c6f2  656 B    *2*
 618020744734  42 hours ago  /bin/sh -c apt-get update &&   apt-get  215.8 MB *2*
 dea1945146b9  7 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]    0 B      *2*
 <missing>     7 weeks ago   /bin/sh -c mkdir -p /run/systemd && ec  7 B      *2*
 <missing>     7 weeks ago   /bin/sh -c sed -i 's/^#\s*\(deb.*unive  2.753 kB *2*
 <missing>     7 weeks ago   /bin/sh -c rm -rf /var/lib/apt/lists/*  0 B      *2*
 <missing>     7 weeks ago   /bin/sh -c set -xe   && echo '#!/bin/s  194.6 kB *2*
 <missing>     7 weeks ago   /bin/sh -c #(nop) ADD file:8f997234193  187.8 MB *2*
 $ docker history sample-node-app.slim                                        *3*
 IMAGE         CREATED       CREATED BY                              SIZE     *4*
 7451554aa807  42 hours ago  /bin/sh -c #(nop)  ENTRYPOINT ["node"   0 B      *4*
 2616067ec78d  42 hours ago  /bin/sh -c #(nop)  EXPOSE 8000/tcp      0 B      *4*
 4ef6d57d3230  42 hours ago  /bin/sh -c #(nop)  ENV PATH=/usr/local  0 B      *4*
 2d82f368c130  42 hours ago  /bin/sh -c #(nop)  WORKDIR /opt/my/ser  0 B      *4*
 0953a87c8e4f  42 hours ago  /bin/sh -c #(nop) COPY dir:36323da1e97  14.02 MB *4*
```

+   ***1*** **在 sample-node-app 镜像上运行 docker history 命令。**

+   ***2*** **此镜像的历史记录显示了创建时每个命令的情况。**

+   ***3*** **在 sample-node-app.slim 镜像上运行 docker history 命令。**

+   ***4*** **slim 容器的历史记录包含更少的命令，包括原始胖镜像中不存在的 COPY 命令。**

上述输出提供了 DockerSlim 所做部分工作的线索。它通过获取最终的文件系统状态，并将该目录作为镜像的最后一层来复制，成功将镜像大小减少到（实际上）单个 14 MB 层。

DockerSlim 生成的其他工件与其最初描述的第二目的相关。它生成一个 seccomp.json 文件（在本例中为 sample-node-app-seccomp.json），该文件可用于限制运行容器的操作。

让我们来看看这个文件的详细内容（此处已编辑，因为它相当长）。

##### 列表 14.7\. 一个 seccomp 配置文件

```
$ SECCOMPFILE=$(ls $(pwd)/.images/*/artifacts/sample-node-app-seccomp.json)  *1*
 $ cat ${SECCOMPFILE}                                                        *2*
 {
"defaultAction": "SCMP_ACT_ERRNO",                                           *3*
   "architectures": [                                                        *4*
   "SCMP_ARCH_X86_64"                                                        *4*
   ],                                                                        *4*
   "syscalls": [                                                             *5*
     {                                                                       *5*
       "name": "capset",                                                     *5*
       "action": "SCMP_ACT_ALLOW"                                            *5*
     },                                                                      *5*
     {                                                                       *5*
       "name": "rt_sigaction",                                               *5*
       "action": "SCMP_ACT_ALLOW"                                            *5*
     },                                                                      *5*
     {                                                                       *5*
       "name": "write",                                                      *5*
       "action": "SCMP_ACT_ALLOW"                                            *5*
     },                                                                      *5*
 [...]                                                                       *5*
    {                                                                        *5*
     "name": "execve",                                                       *5*
       "action": "SCMP_ACT_ALLOW"                                            *5*
     },                                                                      *5*
     {                                                                       *5*
       "name": "getcwd",                                                     *5*
       "action": "SCMP_ACT_ALLOW"                                            *5*
     }                                                                       *5*
   ]
}
```

+   ***1*** **将 seccomp 文件的位置捕获到变量 SECCOMPFILE 中**

+   ***2*** **将此文件内容输出到查看**

+   ***3*** **指定尝试调用任何禁止的系统调用的进程的退出代码**

+   ***4*** **指定此配置文件应应用其上的硬件架构**

+   ***5*** **在此处通过指定 SCMP_ACT_ALLOW 动作对它们进行白名单管理，以控制受控的系统调用。**

最后，您将再次运行带有 seccomp 配置文件的 slim 镜像，并检查它是否按预期工作：

```
$ docker run -p32770:8000 -d \
--security-opt seccomp=/root/docker-slim-bin/.images/${IMAGEID}/artifacts
 /sample-node-app-seccomp.json sample-node-app.slim                   *1*
 4107409b61a03c3422e07973248e564f11c6dc248a6a5753a1db8b4c2902df55        *2*
 $ sleep 10 && curl localhost:3277l                                      *3*
 {"status":"success","info":"yes!!!","service":"node"}                   *4*
```

+   ***1*** **以守护进程运行 slim 镜像，暴露 DockerSlim 在分析阶段暴露的相同端口，并应用 seccomp 配置文件**

+   ***2*** **将容器 ID 输出到终端**

+   ***3*** **重新运行 curl 命令以确认应用程序仍然像以前一样工作**

+   ***4*** **输出与您精简的胖镜像相同。**

**讨论**

这个简单的例子展示了如何不仅减小镜像的大小，还可以缩小其可以执行的操作范围。这是通过删除非必要文件（也在技术 59 中讨论过）以及将其可用的系统调用减少到仅运行应用程序所需的那些来实现的。

在这里“锻炼”应用程序的方法很简单（一个对默认端点的 `curl` 请求）。对于真实的应用程序，你可以采取多种方法来确保你已经覆盖了所有可能性。一种方法是对已知的端点开发一系列测试，另一种方法是使用“模糊器”以自动化的方式向应用程序抛出大量输入（这是找到你的软件中的错误和安全漏洞的一种方法）。最简单的方法是让应用程序运行更长的时间，期望所有需要的文件和系统调用都会被引用。

许多企业级 Docker 安全工具基于这个原则工作，但以更自动化的方式进行。通常，它们允许应用程序运行一段时间，并跟踪哪些系统调用被使用，哪些文件被访问，以及（可能）使用了哪些操作系统功能。基于这些信息——以及可配置的学习期——它们可以确定应用程序的预期行为，并报告任何看似异常的行为。例如，如果攻击者获得了正在运行的容器的访问权限并启动了 bash 二进制文件或打开了意外的端口，这可能会在系统中引发警报。DockerSlim 允许您从一开始就控制这个过程，即使攻击者获得了访问权限，也能减少他们可能能够执行的操作。

另一种考虑缩小应用程序攻击面的方法是限制其功能。这将在技术 93 中介绍。

|  |

**删除构建过程中添加的秘密**

当你在企业环境中构建镜像时，通常需要使用密钥和凭证来检索数据。如果你使用 Dockerfile 构建应用程序，即使在使用后删除，这些秘密通常也会出现在历史记录中。

这可能是一个安全问题：如果有人获得了镜像，他们也可能获得早期层中的秘密。

**问题**

你想从镜像的历史记录中删除一个文件。

**解决方案**

使用 `docker-squash` 从镜像中移除层。

有一些简单的方法可以解决这个问题，从理论上来说是可行的。例如，你可以在使用秘密时将其删除，如下所示。

##### 列表 14.8\. 在层内不留下秘密的粗略方法

```
FROM ubuntu
RUN echo mysecret > secretfile && command_using_secret && rm secretfile
```

这种方法存在一些缺点。它需要将秘密放入 Dockerfile 中的代码，因此它可能以纯文本的形式存储在你的源代码控制中。

为了避免这个问题，您可能需要在源控制中的 .gitignore（或类似）文件中添加该文件，并在构建镜像时将其 `ADD` 到镜像中。这将在一个单独的层中添加文件，无法轻易从生成的镜像中移除。

最后，您可以使用环境变量来存储秘密，但这也会带来安全风险，因为这些变量很容易在非安全的持久存储中设置，例如 Jenkins 作业。在任何情况下，您可能会收到一个用户提供的镜像，并要求您从其中清除秘密。首先，我们将通过一个简单的示例演示这个问题，然后我们将向您展示一种从基础层中删除秘密的方法。

**带有秘密的图像**

以下 Dockerfile 将创建一个镜像，使用名为 secret_file 的文件作为您放入镜像中的某些秘密数据的占位符。

##### 列表 14.9\. 带有秘密的简单 Dockerfile

```
FROM ubuntu
CMD ls /                             *1*
 ADD /secret_file secret_file        *2*
 RUN cat /secret_file                *3*
 RUN rm /secret_file                 *4*
```

+   ***1*** **为了节省一点时间，我们用文件列表命令覆盖了默认命令。这将演示文件是否在历史记录中。**

+   ***2*** **将秘密文件添加到镜像构建中（此文件必须存在于您的当前工作目录中，与 Dockerfile 一起）**

+   ***3*** **将秘密文件作为构建的一部分使用。在这种情况下，我们使用简单的 cat 命令输出文件，但这可以是 git clone 或其他更有用的命令。**

+   ***4*** **删除秘密文件**

现在，您可以构建这个镜像，将生成的镜像命名为 secret_build。

##### 列表 14.10\. 构建带有秘密的简单 Docker 镜像

```
$ echo mysecret > secret_file
$ docker build -t secret_build .
Sending build context to Docker daemon  5.12 kB
Sending build context to Docker daemon
Step 0 : FROM ubuntu
 ---> 08881219da4a
Step 1 : CMD ls /
 ---> Running in 7864e2311699
 ---> 5b39a3cba0b0
Removing intermediate container 7864e2311699
Step 2 : ADD /secret_file secret_file
 ---> a00886ff1240
Removing intermediate container 4f279a2af398
Step 3 : RUN cat /secret_file
 ---> Running in 601fdf2659dd
My secret
 ---> 2a4238c53408
Removing intermediate container 601fdf2659dd
Step 4 : RUN rm /secret_file
 ---> Running in 240a4e57153b
 ---> b8a62a826ddf
Removing intermediate container 240a4e57153b
Successfully built b8a62a826ddf
```

镜像构建完成后，您可以通过使用技术 27 来演示它包含秘密文件。

##### 列表 14.11\. 为每个步骤添加标签并展示带有秘密的层

```
$ x=0; for id in $(docker history -q secret_build:latest);
 do ((x++)); docker tag $id secret_build:step_$x; done      *1*
 $ docker run secret_build:step_3 cat /secret_file'            *2*
 mysecret
```

+   ***1*** **演示秘密文件存在于这个镜像标签中**]

+   ***2*** **按数字顺序标记构建的每个步骤**

**压缩镜像以删除秘密**

您已经看到，即使秘密不在最终的镜像中，秘密也可以保留在镜像的历史记录中。这就是 docker-squash 发挥作用的地方——它移除了中间层，但保留了 Dockerfile 命令（如 `CMD`、`PORT`、`ENV` 等）以及您历史记录中的原始基础层。

以下列表下载、安装并使用 `docker-squash` 来比较压缩前后的镜像。

##### 列表 14.12\. 使用 `docker_squash` 减少镜像的层

```
$ wget -qO- https://github.com/jwilder/docker-squash/releases/download
 /v0.2.0/docker-squash-linux-amd64-v0.2.0.tar.gz | \
  tar -zxvf -  && mv docker-squash /usr/local/bin                          *1*
 $ docker save secret_build:latest | \                                     *2*
   docker-squash -t secret_build_squashed | \                              *2*
   docker load                                                             *2*
 $ docker history secret_build_squashed                                    *3*
 IMAGE         CREATED        CREATED BY                          SIZE
ee41518cca25  2 seconds ago  /bin/sh -c #(nop) CMD ["/bin/sh" "  0 B
b1c283b3b20a  2 seconds ago  /bin/sh -c #(nop)  CMD ["/bin/bash  0 B
f443d173e026  2 seconds ago  /bin/sh -c #(squash) from 93c22f56  2.647 kB
93c22f563196  2 weeks ago    /bin/sh -c #(nop) ADD file:7529d28  128.9 MB
$ docker history secret_build                                              *4*
 IMAGE         CREATED        CREATED BY                          SIZE
b8a62a826ddf  3 seconds ago  /bin/sh -c rm /secret_file          0 B
2a4238c53408  3 seconds ago  /bin/sh -c cat /secret_file         0 B
a00886ff1240  9 seconds ago  /bin/sh -c #(nop) ADD file:69e77f6  10 B
5b39a3cba0b0  9 seconds ago  /bin/sh -c #(nop) CMD ["/bin/sh" "  0 B
08881219da4a  2 weeks ago    /bin/sh -c #(nop)  CMD ["/bin/bash  0 B
6a4ec4bddc58  2 weeks ago    /bin/sh -c mkdir -p /run/systemd &  7 B
98697477f76a  2 weeks ago    /bin/sh -c sed -i 's/^#\s*\(deb.*u  1.895 kB
495ec797e6ba  2 weeks ago    /bin/sh -c rm -rf /var/lib/apt/lis  0 B
e3aa81f716f6  2 weeks ago    /bin/sh -c set -xe && echo '#!/bin  745 B
93c22f563196  2 weeks ago    /bin/sh -c #(nop) ADD file:7529d28  128.9 MB
$ docker run secret_build_squashed ls /secret_file                         *5*
 ls: cannot access '/secret_file': No such file or directory
$ docker run f443d173e026 ls /secret_file                                  *6*
 ls: cannot access '/secret_file': No such file or directory
```

+   ***1*** **安装 docker-squash. (您可能需要参考[`github.com/jwilder/docker-squash`](https://github.com/jwilder/docker-squash)以获取最新的安装说明。)**

+   ***2*** **将镜像保存到 TAR 文件中，然后加载结果镜像，将其标记为“secret_build_squashed”**

+   ***3*** **压缩后的镜像的历史记录中没有 secret_file 的记录。**

+   ***4*** **原始镜像中仍然包含 secret_file。**

+   ***5*** **演示 secret_file 不在压缩后的镜像中**

+   ***6*** **演示了 secret_file 不在压缩镜像的“压缩”层中**

**关于“缺失”镜像层的说明**

Docker 在 1.10 版本中改变了层的基本性质。从那时起，下载的镜像在历史中显示为“<missing>”。这是预期的，这是由于 Docker 对镜像历史所做的更改，以提高镜像的安全性。

你仍然可以通过 `docker save` 镜像并从该 TAR 文件中提取 TAR 文件来获取你已下载的层的内容。以下是一个示例会话，它为已下载的 Ubuntu 镜像执行了此操作。

##### 列表 14.13\. “缺失”的层在下载的镜像中

```
$ docker history ubuntu                                                        *1*
 IMAGE         CREATED      CREATED BY                               SIZE
104bec311bcd  2 weeks ago  /bin/sh -c #(nop)  CMD ["/bin/bash"]     0 B
<missing>     2 weeks ago  /bin/sh -c mkdir -p /run/systemd && ech  7 B
<missing>     2 weeks ago  /bin/sh -c sed -i 's/^#\s*\(deb.*univer  1.9 kB
<missing>     2 weeks ago  /bin/sh -c rm -rf /var/lib/apt/lists/*   0 B
<missing>     2 weeks ago  /bin/sh -c set -xe   && echo '#!/bin/sh  745 B
<missing>     2 weeks ago  /bin/sh -c #(nop) ADD file:7529d28035b4  129 MB
$ docker save ubuntu | tar -xf -                                               *2*
 $ find . | grep tar$                                                          *3*
 ./042e55060780206b2ceabe277a8beb9b10f48262a876fd21b495af318f2f2352/layer.tar
./1037e0a8442d212d5cc63d1bc706e0e82da0eaafd62a2033959cfc629f874b28/layer.tar
./25f649b30070b739bc2aa3dd877986bee4de30e43d6260b8872836cdf549fcfc/layer.tar
./3094e87864d918dfdb2502e3f5dc61ae40974cd957d5759b80f6df37e0e467e4/layer.tar
./41b8111724ab7cb6246c929857b0983a016f11346dcb25a551a778ef0cd8af20/layer.tar
./4c3b7294fe004590676fa2c27a9a952def0b71553cab4305aeed4d06c3b308ea/layer.tar
./5d1be8e6ec27a897e8b732c40911dcc799b6c043a8437149ab021ff713e1044f/layer.tar
./a594214bea5ead6d6774f7a09dbd7410d652f39cc4eba5c8571d5de3bcbe0057/layer.tar
./b18fcc335f7aeefd87c9d43db2888bf6ea0ac12645b7d2c33300744c770bcec7/layer.tar
./d899797a09bfcc6cb8e8a427bb358af546e7c2b18bf8e2f7b743ec36837b42f2/layer.tar
./ubuntu.tar
$ tar -tvf
 ./4c3b7294fe004590676fa2c27a9a952def0b71553cab4305aeed4d06c3b308ea
 /layer.tar
drwxr-xr-x  0 0      0           0 15 Dec 17:45 etc/
drwxr-xr-x  0 0      0           0 15 Dec 17:45 etc/apt/
-rw-r--r--  0 0      0        1895 15 Dec 17:45 etc/apt/sources.list
```

+   ***1*** **使用 docker history 命令显示 Ubuntu 镜像的层历史**

+   ***2*** **使用 docker save 命令输出镜像层的 TAR 文件，该文件直接通过管道传输到 tar 并提取**

+   ***3*** **演示了 TAR 文件只包含该层内的文件更改**

**讨论**

虽然在意图上与 技术 52 有一定相似性，但使用专用工具在最终结果上存在一些显著差异。在前面的解决方案中，你可以看到像 `CMD` 这样的元数据层已经被保留，而之前关于这个主题的技术会完全丢弃它们，因此你需要通过另一个 Dockerfile 手动重新创建这些元数据层。

这种行为意味着 docker-squash 工具可以在镜像到达注册表时自动清理镜像，如果你不信任用户在镜像构建中使用秘密数据——它们都应该正常工作。

话虽如此，你应该警惕你的用户将秘密信息放入任何元数据层中——特别是环境变量是一个威胁，并且可能会在最终镜像中被保留。

|  |

**OpenShift：一个应用程序平台即服务**

OpenShift 是由 Red Hat 管理的一个产品，它允许组织以服务的形式运行应用程序平台（aPaas）。它为应用程序开发团队提供了一个平台，可以在其中运行代码，而无需关心硬件细节。该产品的第 3 版是在 Go 语言中从头开始重写的，使用 Docker 作为容器技术，Kubernetes 和 etcd 进行编排。在此基础上，Red Hat 还添加了企业功能，使其更容易在企业和安全重点环境中部署。

尽管 OpenShift 有许多我们可以讨论的功能，但在这里我们将使用它作为管理安全性的手段，通过剥夺用户直接运行 Docker 的能力，但保留使用 Docker 的好处。

OpenShift 既可以作为企业支持的产品提供，也可以作为一个名为 Origin 的开源项目提供，该项目由 [`github.com/openshift/origin`](https://github.com/openshift/origin) 维护。

**问题**

你希望管理不受信任的用户调用 `docker run` 的安全风险。

**解决方案**

使用 aPaaS 工具通过代理接口管理和调解与 Docker 的交互。

aPaaS 有许多好处，但在这里我们将关注其管理用户权限并在用户代表下运行 Docker 容器的能力，为运行 Docker 容器的用户提供一个安全的审计点。

这为什么很重要？使用这个 aPaaS 的用户没有直接访问 `docker` 命令的权限，因此他们不能在不绕过 OpenShift 提供的安全性的情况下造成任何损害。例如，容器默认由非 root 用户部署，克服这一点需要管理员授予的权限。如果你不信任你的用户，使用 aPaaS 是一种有效的方式，让他们能够访问 Docker。


##### 提示

aPaaS 为用户提供按需启动应用以进行开发、测试或生产的能力。Docker 是这些服务的自然选择，因为它提供了一种可靠且隔离的应用交付格式，允许运维团队处理部署的细节。


简而言之，OpenShift 基于 Kubernetes（见技术 88），但增加了功能，以提供完整的 aPaaS。这些附加功能包括

+   用户管理

+   权限管理

+   配额

+   安全上下文

+   路由

**安装 OpenShift**

OpenShift 安装的完整概述超出了本书的范围。如果您想使用我们维护的 Vagrant 进行自动化安装，请参阅[`github.com/docker-in-practice/shutit-openshift-origin`](https://github.com/docker-in-practice/shutit-openshift-origin)。如果您需要安装 Vagrant 的帮助，请参阅附录 C。

其他选项，如仅 Docker 安装（单节点）或完整的手动构建，都是可用的，并在 OpenShift Origin 代码库中有文档说明[`github.com/openshift/origin.git`](https://github.com/openshift/origin.git)。


##### 提示

OpenShift Origin 是 OpenShift 的上游版本。*上游*意味着它是 Red Hat 为 OpenShift 取得变更的代码库，它是 Red Hat 的支持产品。Origin 是开源的，任何人都可以使用和贡献，但 Red Hat 精选的版本作为 OpenShift 出售和支持。上游版本通常更前沿但更不稳定。


**一个 OpenShift 应用**

在这个技术中，我们将通过 OpenShift 网页界面展示一个创建、构建、运行和访问应用的简单示例。该应用将是一个基本的 NodeJS 应用，它提供了一个简单的网页。

应用将使用 Docker、Kubernetes 和 S2I。Docker 用于封装构建和部署环境。源到镜像（S2I）构建方法是由 Red Hat 在 OpenShift 中用于构建 Docker 容器的一种技术，而 Kubernetes 用于在 OpenShift 集群上运行应用。

**登录**

要开始，从 shutit-openshift-origin 文件夹运行 `./run.sh`，然后导航到 https://localhost:8443，绕过所有安全警告。你会看到如图 图 14.3 所示的登录页面。注意，如果你使用 Vagrant 安装，你需要在你的虚拟机中启动一个网络浏览器。（有关在虚拟机中获得 GUI 的帮助，请参阅 附录 C）。

##### 图 14.3\. OpenShift 登录页面

![图 14.03](img/14fig03_alt.jpg)

使用任何密码登录为 `hal-1`。

**构建 NodeJS 应用**

你现在已以开发者的身份登录到 OpenShift（见图 图 14.4）。

##### 图 14.4\. OpenShift 项目页面

![图 14.04](img/14fig04_alt.jpg)

通过点击创建来创建一个项目。填写表格，如图 图 14.5 所示。然后再次点击创建。

##### 图 14.5\. OpenShift 项目创建页面

![图 14.05](img/14fig05_alt.jpg)

一旦项目设置完成，再次点击创建，并输入建议的 GitHub 仓库 ([`github.com/openshift/nodejs-ex`](https://github.com/openshift/nodejs-ex))，如图 图 14.6 所示。

##### 图 14.6\. OpenShift 项目源页面

![图 14.06](img/14fig06_alt.jpg)

点击下一步，你将看到一系列构建器镜像的选择，如图 图 14.7 所示。构建镜像定义了代码将构建的上下文。选择 NodeJS 构建器镜像。

##### 图 14.7\. OpenShift 构建器镜像选择页面

![图 14.07](img/14fig07_alt.jpg)

现在填写表格，如图 图 14.8 所示。在滚动表格时，在页面底部点击 NodeJS 的创建。

##### 图 14.8\. OpenShift NodeJS 模板表单

![图 14.08](img/14fig08_alt.jpg)

几分钟后，你应该会看到一个类似于 图 14.9 的屏幕。

##### 图 14.9\. OpenShift 构建开始页面

![图 14.09](img/14fig09_alt.jpg)

几分钟后，如果你向下滚动，你会看到构建已经开始，如图 图 14.10 所示。

##### 图 14.10\. OpenShift 构建信息窗口

![图 14.10](img/14fig10_alt.jpg)

| |
| --- |

##### 小贴士

在 OpenShift 的早期版本中，构建有时不会自动开始。如果这种情况发生，几分钟后点击开始构建按钮。

| |
| --- |

一段时间后，你会看到应用正在运行，如图 图 14.11 所示。

##### 图 14.11\. 应用运行页面

![图 14.11](img/14fig11_alt.jpg)

通过点击浏览和 Pods，你可以看到 pod 已经部署，如 图 14.12 所示。

##### 图 14.12\. OpenShift pods 列表

![图 14.12](img/14fig12_alt.jpg)

| |
| --- |

##### 小贴士

查看 技术 88 了解 pod 的解释。

| |
| --- |

如何访问你的 pod？如果你查看服务标签（见图 图 14.13），你会看到一个 IP 地址和端口号来访问。

##### 图 14.13\. OpenShift NodeJS 应用服务详情

![图 14.13](img/14fig13_alt.jpg)

将你的浏览器指向该地址，哇，你将拥有你的 NodeJS 应用，如图 图 14.14 所示。

##### 图 14.14\. NodeJS 应用程序登录页面

![](img/14fig14_alt.jpg)

**讨论**

让我们回顾一下我们在这里所取得的成果，以及为什么这对安全性很重要。

从用户的角度来看，他们登录了一个 Web 应用程序，并使用基于 Docker 的技术部署了一个应用程序，而没有接近 Dockerfile 或 `docker run` 命令。

OpenShift 的管理员可以

+   控制用户访问

+   按项目限制资源使用

+   集中提供资源

+   默认确保代码以非特权状态运行

这比直接给用户 `docker run` 访问权限要安全得多。

如果您想在此基础上构建应用程序并了解一个 aPaaS 如何促进迭代方法，您可以分叉 Git 仓库，更改该分叉仓库中的代码，然后创建一个新的应用程序。我们在这里就是这样做的：[`github.com/docker-in-practice/nodejs-ex`](https://github.com/docker-in-practice/nodejs-ex)。

要了解更多关于 OpenShift 的信息，请访问 [`www.openshift.org`](http://www.openshift.org)。

|  |

**使用安全选项**

您已经在前面的技术中看到，默认情况下，您在 Docker 容器中拥有 root 权限，并且这个用户与宿主机的 root 用户相同。为了减轻这一点，我们向您展示了如何降低此用户的 root 能力，即使它逃出了容器，内核也不会允许此用户执行某些操作。

但您可以更进一步。通过使用 Docker 的安全选项标志，您可以保护宿主机的资源免受容器内执行的操作的影响。这限制了容器只能影响宿主机授予其权限的资源。

**问题**

您希望保护您的宿主免受容器操作的影响。

**解决方案**

使用 SELinux 对您的容器施加约束。

在这里，我们将使用 SELinux 作为我们的内核支持的强制访问控制（MAC）工具。SELinux 大约是行业标准，最有可能被特别关注安全的组织使用。它最初由 NSA 开发，用于保护他们的系统，后来开源。它在基于 Red Hat 的系统中作为标准使用。

SELinux 是一个很大的主题，所以我们不能在这本书中深入探讨。我们将向您展示如何编写和执行一个简单的策略，以便您可以了解它是如何工作的。如果您需要，您可以进一步探索并进行实验。


##### 提示

Linux 中的强制访问控制（MAC）工具强制执行超出您可能习惯的标准安全规则。简而言之，它们不仅确保了文件和进程的读写执行等常规规则得到执行，而且还可以在内核级别应用更细粒度的规则。例如，MySQL 进程可能仅被允许在特定目录下写入文件，例如 /var/lib/mysql。基于 Debian 的系统的等效标准是 AppArmor。


这种技术假设你有一个启用了 SELinux 的主机。这意味着你必须首先安装 SELinux（假设它尚未安装）。如果你正在运行 Fedora 或其他基于 Red Hat 的系统，你很可能已经安装了它。

要确定你是否启用了 SELinux，运行命令 `sestatus`：

```
# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28
```

输出的第一行将告诉你 SELinux 是否已启用。如果该命令不可用，则表示你的主机上未安装 SELinux。

你还需要有相关的 SELinux 政策创建工具可用。例如，在支持 yum 的机器上，你需要运行 `yum -y install selinux-policy -devel`。

**Vagrant 机器上的 SELinux**

如果你没有 SELinux 并且希望它为你构建，你可以使用 ShutIt 脚本在你的主机机器内部构建一个带有 Docker 和 SELinux 预安装的虚拟机。它所做的工作在图 14.15 中进行了高层次解释。

##### 图 14.15\. 配置 SELinux 虚拟机的脚本

![图片](img/14fig15_alt.jpg)


##### 提示

ShutIt 是一个通用的 shell 自动化工具，我们创建它是为了克服 Dockerfile 的一些限制。如果你想了解更多关于它的信息，请参阅 GitHub 页面：[`ianmiell.github.io/shutit`](http://ianmiell.github.io/shutit)。


图 14.5 识别了设置策略所需的步骤。该脚本将执行以下操作：

1.  设置 VirtualBox

1.  启动合适的 Vagrant 镜像

1.  登录到虚拟机

1.  确保 SELinux 的状态正确

1.  安装 Docker 的最新版本

1.  安装 SELinux 政策开发工具

1.  给你一个 shell

这里是设置和运行它的命令（已在 Debian 和基于 Red Hat 的发行版上测试）：

##### 列表 14.14\. 安装 ShutIt

```
sudo su -                                                     *1*
 apt-get install -y git python-pip docker.io || \
yum install -y git python-pip docker.io                       *2*
 pip install shutit                                           *3*
 git clone https://github.com/ianmiell/docker-selinux.git     *4*
 cd docker-selinux                                            *4*
 shutit build --delivery bash \                               *5*
 -s io.dockerinpractice.docker_selinux.docker_selinux \
  compile_policy no                                           *6*
```

+   ***1*** **确保在开始运行之前你是 root 用户**

+   ***2*** **确保主机上安装了所需的软件包**

+   ***3*** **安装 ShutIt**

+   ***4*** **克隆 SELinux ShutIt 脚本并进入其目录**

+   ***5*** **运行 ShutIt 脚本。“--delivery bash”表示命令在 bash 中执行，而不是通过 SSH 或 Docker 容器执行。**

+   ***6*** **配置脚本不编译 SELinux 政策，因为我们将会手动完成**

运行此脚本后，你应该最终看到如下输出：

```
Pause point:
Have a shell:
You can now type in commands and alter the state of the target.
Hit return to see the prompt
Hit CTRL and ] at the same time to continue with build

Hit CTRL and u to save the state
```

现在，你有一个运行在带有 SELinux 的虚拟机内部的 shell。如果你输入 `sestatus`，你会看到 SELinux 以许可模式启用（如列表 14.14 所示）。要返回到主机的 shell，请按 Ctrl-]。

**编译 SELinux 政策**

无论你是否使用了 ShutIt 脚本，我们假设你现在有一个启用了 SELinux 的主机。输入 `sestatus` 以获取状态摘要。

##### 列表 14.15\. 安装并启用后 SELinux 的状态

```
# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28
```

在这种情况下，我们处于允许模式，这意味着 SELinux 正在记录日志中的安全违规，但不会强制执行它们。这对于安全测试新策略而不会使系统不可用是有益的。要将 SELinux 状态更改为允许模式，请以 root 身份输入`setenforce Permissive`。如果您由于安全原因无法在主机上执行此操作，请不要担心；有一个选项可以将策略设置为允许模式，如列表 14.15 中概述。


##### 注意

如果您在主机上自行安装 SELinux 和 Docker，请确保 Docker 守护进程已将`--selinux-enabled`设置为标志。您可以使用`ps -ef | grep 'docker -d.*--selinux-enabled'`来检查此设置，它应该在输出中返回一个匹配的进程。


为您的策略创建一个文件夹并进入它。然后以 root 身份创建一个策略文件，内容如下，命名为 docker_apache.te。此策略文件包含我们将尝试应用的策略。

##### 列表 14.16\. 创建 SELinux 策略

```
mkdir -p /root/httpd_selinux_policy && >
cd /root/httpd_selinux_policy                                              *1*
 cat > docker_apache.te << END                                             *2*
 policy_module(docker_apache,1.0)                                          *3*
 virt_sandbox_domain_template(docker_apache)                               *4*
 allow docker_apache_t self: capability { chown dac_override kill setgid >
setuid net_bind_service sys_chroot sys_nice >
sys_tty_config } ;                                                         *5*
 allow docker_apache_t self:tcp_socket >
create_stream_socket_perms;                                                *6*
 allow docker_apache_t self:udp_socket >                                   *6*
create_socket_perms;                                                       *6*
 corenet_tcp_bind_all_nodes(docker_apache_t)                               *6*
 corenet_tcp_bind_http_port(docker_apache_t)                               *6*
 corenet_udp_bind_all_nodes(docker_apache_t)                               *6*
 corenet_udp_bind_http_port(docker_apache_t)                               *6*
 sysnet_dns_name_resolve(docker_apache_t)                                  *7*
 #permissive docker_apache_t                                               *8*
 END                                                                       *9*
```

+   ***1*** **创建一个文件夹来存储策略文件，并进入该文件夹**

+   ***2*** **创建将作为“here”文档编译的策略文件**

+   ***3*** **使用 policy_module 指令创建名为 docker_apache 的 SELinux 策略模块**

+   ***4*** **使用提供的模板创建 docker_apache_t SELinux 类型，该类型可以作为 Docker 容器运行。此模板为 docker_apache SELinux 域提供了运行所需的最少权限。我们将添加这些权限以创建一个有用的容器环境。**

+   ***5*** **Apache 网络服务器需要这些功能才能运行；使用 allow 指令在此处添加它们。**

+   ***6*** **这些 allow 和 corenet 规则允许容器在网络中监听 Apache 端口。**

+   ***7*** **使用 sysnet 指令允许 DNS 服务器解析**

+   ***8*** **可选地使 docker_apache_t 类型为允许模式，即使主机正在执行 SELinux 策略，此策略也不会被强制执行。如果你无法设置主机的 SELinux 模式，请使用此选项。**

+   ***9*** **终止“here”文档，将其写入磁盘**


##### 提示

关于前面的权限的更多信息，以及探索其他权限，您可以安装 selinux-policy-doc 包，并使用浏览器浏览位于 file:///usr/share/doc-base/selinux-policy-doc/html/index.html 的文档。文档也在线上可用，网址为[`oss.tresys.com/docs/refpolicy/api/`](http://oss.tresys.com/docs/refpolicy/api/)。


现在您将编译此策略，并看到您的应用程序在强制模式下无法启动。然后您将重新启动它以检查违规行为并在稍后进行纠正：

```
$ make -f /usr/share/selinux/devel/Makefile \
docker_apache.te                                                  *1*
 Compiling targeted docker_apache module
/usr/bin/checkmodule:  loading policy configuration from >
tmp/docker_apache.tmp
/usr/bin/checkmodule:  policy configuration loaded
/usr/bin/checkmodule:  writing binary representation (version 17) >
to tmp/docker_apache.mod
Creating targeted docker_apache.pp policy package
rm tmp/docker_apache.mod tmp/docker_apache.mod.fc
$ semodule -i docker_apache.pp                                    *2*
 $ setenforce Enforcing                                           *3*
 $ docker run -ti --name selinuxdock >
--security-opt label:type:docker_apache_t httpd                   *4*
 Unable to find image 'httpd:latest' locally
latest: Pulling from library/httpd
2a341c7141bd: Pull complete
[...]
Status: Downloaded newer image for httpd:latest
permission denied
Error response from daemon: Cannot start container >
650c446b20da6867e6e13bdd6ab53f3ba3c3c565abb56c4490b487b9e8868985: >
[8] System error: permission denied
$ docker rm -f selinuxdock                                        *5*
 selinuxdock
$ setenforce Permissive                                           *6*
 $ docker run -d --name selinuxdock >
--security-opt label:type:docker_apache_t httpd                   *7*
```

+   ***1*** **将 docker_apache.te 文件编译为具有.pps 后缀的二进制 SELinux 模块**

+   ***2*** **安装模块**

+   ***3*** **将 SELinux 模式设置为“enforcing”**

+   ***4*** **以守护进程方式运行 httpd 镜像，应用你在 SELinux 模块中定义的 docker_apache_t 安全标签类型。此命令应失败，因为它违反了 SELinux 安全配置。**

+   ***5*** **删除新创建的容器**

+   ***6*** **将 SELinux 模式设置为“permissive”以允许应用程序启动**

+   ***7*** **以守护进程方式运行 httpd 镜像，应用你在 SELinux 模块中定义的 docker_apache_t 安全标签类型。此命令应成功运行。**

**检查违规**

到目前为止，你已经创建了一个 SELinux 模块并将其应用到你的主机上。因为在这个主机上 SELinux 的强制模式设置为 permissive，所以那些在强制模式下会被禁止的操作允许在审计日志中记录下来。你可以通过运行以下命令来检查这些消息：

```
$ grep -w denied /var/log/audit/audit.log
type=AVC msg=audit(1433073250.049:392): avc:  >                        *1*
 denied  { transition } for >                                          *2*
 pid=2379 comm="docker" >                                              *3*
 path="/usr/local/bin/httpd-foreground" dev="dm-1" ino=530204 >        *4*
 scontext=system_u:system_r:init_t:s0 >
tcontext=system_u:system_r:docker_apache_t:s0:c740,c787 >              *5*
 tclass=process                                                        *6*
 type=AVC msg=audit(1433073250.049:392): avc:  denied  { write } for  >
pid=2379 comm="httpd-foregroun" path="pipe:[19550]" dev="pipefs" >
ino=19550 scontext=system_u:system_r:docker_apache_t:s0:c740,c787 >
tcontext=system_u:system_r:init_t:s0 tclass=fifo_file
type=AVC msg=audit(1433073250.236:394): avc:  denied  { append } for  >
pid=2379 comm="httpd" dev="pipefs" ino=19551 >
scontext=system_u:system_r:docker_apache_t:s0:c740,c787 >
tcontext=system_u:system_r:init_t:s0 tclass=fifo_file
type=AVC msg=audit(1433073250.236:394): avc:  denied  { open } for  >
pid=2379 comm="httpd" path="pipe:[19551]" dev="pipefs" ino=19551 >
scontext=system_u:system_r:docker_apache_t:s0:c740,c787 >
tcontext=system_u:system_r:init_t:s0 tclass=fifo_file
[...]
```

+   ***1*** **审计日志中的消息类型始终为 AVC，表示 SELinux 违规，时间戳以自纪元（定义为 1970 年 1 月 1 日）以来的秒数给出。**

+   ***2*** **拒绝的操作类型显示在花括号中。**

+   ***3*** **触发违规的命令的进程 ID 和名称**

+   ***4*** **目标文件的路径、设备和 inode**

+   ***5*** **目标的安全上下文**

+   ***6*** **目标对象的类别**

呼吁！这里有太多的术语，我们没有时间教你可能需要知道的所有关于 SELinux 的知识。如果你想了解更多，一个好的开始是查看 Red Hat 的 SELinux 文档：[`access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Deployment_Guide/ch-selinux.html`](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Deployment_Guide/ch-selinux.html)。

目前，你需要检查违规是否没有不寻常之处。什么可能看起来不寻常？如果一个应用程序尝试打开你未预期的端口或文件，你可能会三思而后行：使用新的 SELinux 模块修补这些违规。

在这种情况下，我们很高兴 httpd 可以写入管道。我们已经确定这是 SELinux 阻止的原因，因为提到的“拒绝”操作是对于 VM 上 pipefs 文件的`append`、`write`和`open`。

**修补 SELinux 违规**

一旦你确定你看到的违规是可以接受的，有一些工具可以自动生成你需要应用的政策文件，因此你不需要自己编写一个，避免痛苦和风险。以下示例使用 audit2allow 工具来实现这一点。

##### 列表 14.17. 创建新的 SELinux 策略

```
mkdir -p /root/selinux_policy_httpd_auto       *1*
 cd /root/selinux_policy_httpd_auto
audit2allow -a -w                              *2*
 audit2allow -a -M newmodname create policy    *3*
 semodule -i newmodname.pp                     *4*
```

+   ***1*** **创建一个新的文件夹来存储新的 SELinux 模块**

+   ***2*** **使用 audit2allow 工具显示从读取审计日志生成的策略。再次审查以确保其合理。**

+   ***3*** **使用-M 标志和所选模块的名称创建你的模块**

+   ***4*** **从新创建的 .pp 文件中安装模块**

重要的是要理解我们创建的这个新的 SELinux 模块“包含”（或“需要”）并修改了我们之前创建的模块，通过引用并添加权限到 docker_apache_t 类型。如果你选择，你可以将这两个合并成一个完整且独立的策略，在单个 .te 文件中。

**测试你的新模块**

现在你已经安装了新模块，你可以尝试重新启用 SELinux 并重新启动容器。

| |
| --- |

##### 小贴士

如果你之前无法将主机设置为宽容模式（并且你已将已删除的行添加到原始 docker_apache.te 文件中），则在继续之前重新编译并重新安装原始 docker_apache.te 文件（已删除宽容行）。

| |
| --- |

##### 列表 14.18. 以 SELinux 限制启动容器

```
docker rm -f selinuxdock
setenforce Enforcing
docker run -d --name selinuxdock \
--security-opt label:type:docker_apache_t httpd
docker logs selinuxdock
grep -w denied /var/log/audit/audit.log
```

审计日志中不应出现新的错误。你的应用程序已在这个 SELinux 环境的上下文中启动。

**讨论**

SELinux 以其复杂和难以管理而闻名，最常听到的抱怨是它更常被关闭而不是调试。这几乎不安全。尽管 SELinux 的细微之处确实需要付出极大的努力才能掌握，但我们希望这项技术已经向你展示了如何创建一个安全专家可以审查——如果 Docker 不符合出厂设置，则理想情况下可以批准——的东西。

| |
| --- |

### 摘要

+   你可以使用功能来细粒度地控制容器内 root 的权限。

+   你可以使用 HTTP 通过 Docker API 对人员进行身份验证。

+   Docker 内置了对使用证书进行 API 加密的支撑。

+   SELinux 是一种经过良好测试的方法，可以降低容器以 root 身份运行的风险。

+   作为一种服务（aPaaS）的应用程序平台可以用来控制对 Docker 运行时的访问。

## 第十五章. 一帆风顺：在生产环境中运行 Docker

| |
| --- |

**本章涵盖**

+   你的日志容器输出的选项

+   监控运行中的容器

+   管理你的容器资源使用

+   使用 Docker 的功能来帮助管理传统的系统管理员任务

| |
| --- |

在本章中，我们将涵盖在生产环境中运行时出现的一些主题。在生产环境中运行 Docker 是一个很大的主题，Docker 的生产使用仍然是一个不断发展的领域。许多主要工具都处于早期开发阶段，并且在我们撰写本书的第一版和第二版时正在发生变化。

在本章中，我们将专注于向你展示一些当你从易变环境过渡到稳定环境时应考虑的关键事项。

### 15.1. 监控

当你在生产环境中运行 Docker 时，你首先想要考虑的一件事是如何跟踪和衡量你的容器正在做什么。在本节中，你将学习如何获取你运行中的容器日志活动和性能的操作视图。

这仍然是 Docker 生态系统的一个发展中的方面，但一些工具和技术正在成为比其他更主流的选择。我们将探讨将应用程序日志重定向到主机的 syslog，将`docker logs`命令的输出重定向到单个位置，以及 Google 的面向容器的性能监控工具 cAdvisor。


**将容器的日志记录到主机的 syslog**

Linux 发行版通常运行一个 syslog 守护进程。这个守护进程是系统日志功能的客户端部分——应用程序将消息发送到这个守护进程，以及元数据，如消息的重要性，守护进程将决定将消息保存到何处（如果有的话）。这个功能被各种应用程序使用，从网络连接管理器到内核本身在遇到错误时输出信息。

由于它非常可靠且广泛使用，因此您自己编写的应用程序记录到 syslog 是合理的。不幸的是，一旦您将应用程序容器化（因为默认情况下容器中没有 syslog 守护进程），这将停止工作。如果您决定在所有容器中启动 syslog 守护进程，您需要访问每个单独的容器来检索日志。

**问题**

您希望在您的 Docker 主机上集中捕获 syslogs。

**解决方案**

运行一个充当 Docker 容器 syslog 守护进程的服务容器。

这种技术的基本思想是运行一个服务容器，该容器运行 syslog 守护进程，并通过主机的文件系统共享日志接触点(/dev/log)。日志本身可以通过查询 syslog Docker 容器来检索，并存储在卷中。

图 15.1 说明了主机文件系统上的/tmp/syslogdev 如何用作主机上所有 syslog 操作的接触点。日志容器挂载并写入其 syslog 到该位置，而 syslogger 容器汇总所有这些输入。

##### 图 15.1\. Docker 容器的集中 syslog 概述

![](img/15fig01_alt.jpg)


##### 小贴士

syslog 守护进程是在服务器上运行的一个进程，它收集和管理发送到中央文件的消息，这通常是一个 Unix 域套接字。它通常使用/dev/log 作为接收日志消息的文件，并将日志输出到/var/log/syslog。


使用这个简单的 Dockerfile 可以创建 syslogger 容器。

##### 列表 15.1\. 构建 syslogger 容器

```
FROM ubuntu:14.043
RUN apt-get update && apt-get install rsyslog     *1*
 VOLUME /dev                                      *2*
 VOLUME /var/log                                  *3*
 CMD rsyslogd -n                                  *4*
```

+   ***1*** **安装 rsyslog 包，这使得 rsyslogd 守护进程程序可用。“r”代表“可靠”。**

+   ***2*** **创建一个用于与其他容器共享的 /dev 卷**

+   ***3*** **创建一个 /var/log 卷，以便 syslog 文件可以持久化**

+   ***4*** **在启动时运行 rsyslogd 进程**

接下来，您构建容器，使用 syslogger 标签标记它，并运行它：

```
docker build -t syslogger .
docker run --name syslogger -d -v /tmp/syslogdev:/dev syslogger
```

你将容器的/dev 文件夹绑定挂载到主机的/tmp/syslogdev 文件夹，这样你就可以将/dev/log 套接字作为卷挂载到每个容器中，正如你很快就会看到的。容器将继续在后台运行，读取任何来自/dev/log 文件的消息并处理它们。

在主机上，你现在会看到 syslog 容器的/dev 文件夹已经挂载到主机的/tmp/syslogdev 文件夹：

```
$ ls -1 /tmp/syslogdev/
fd
full
fuse
kcore
log
null
ptmx
random
stderr
stdin
stdout
tty
urandom
zero
```

对于这个演示，我们将启动 100 个守护进程容器，它们将从 0 到 100 记录自己的启动顺序到 syslog，使用`logger`命令。然后，你可以通过在主机上运行`docker exec`来查看 syslogger 容器的 syslog 文件，从而看到这些消息。

首先，启动容器。

##### 列表 15.2\. 启动日志容器

```
for d in {1..100}
do
    docker run -d -v /tmp/syslogdev/log:/dev/log ubuntu logger hello_$d
done
```

之前的卷挂载将容器的 syslog 端点(/dev/log)链接到主机上的/tmp/syslogdev/log 文件，该文件反过来映射到 syslogger 容器的/dev/log 文件。通过这种连接，所有 syslog 输出都发送到同一个文件。

当完成时，你会看到类似以下（编辑后）的输出：

```
$ docker exec -ti syslogger tail -f /var/log/syslog
May 25 11:51:25 f4fb5d829699 logger: hello
May 25 11:55:15 f4fb5d829699 logger: hello_1
May 25 11:55:15 f4fb5d829699 logger: hello_2
May 25 11:55:16 f4fb5d829699 logger: hello_3
[...]
May 25 11:57:38 f4fb5d829699 logger: hello_97
May 25 11:57:38 f4fb5d829699 logger: hello_98
May 25 11:57:39 f4fb5d829699 logger: hello_99
```

如果你愿意，可以使用修改后的`exec`命令来存档这些 syslogs。例如，你可以运行以下命令来获取 5 月 25 日第 11 小时的所有日志存档到一个压缩文件中：

```
$ docker exec syslogger bash -c "cat /var/log/syslog | \
grep '^May 25 11'" | xz - > /var/log/archive/May25_11.log.xz
```


##### 注意

为了让消息显示在中央 syslog 容器中，你的程序需要记录到 syslog。我们通过运行`logger`命令来确保这一点，但你的应用程序也应该这样做才能正常工作。大多数现代日志方法都有一种写入本地可见 syslog 的方法。


**讨论**

你可能会想知道如何使用这种技术区分不同容器的日志消息。这里有几个选项。你可以更改应用程序的日志输出以输出容器的主机名，或者你可以查看下一个技术，让 Docker 为你做这项繁重的工作。


##### 注意

这种技术与下一个使用 Docker syslog 驱动的技术看起来很相似，但它不同。这种技术将容器运行进程的输出作为`docker logs`命令的输出，而下一个技术接管了`logs`命令，使得这种技术变得冗余。

|  |
|  |

**记录 Docker 日志输出**

正如你所见，Docker 提供了一个基本的日志系统，它捕获了容器启动命令的输出。如果你是一个在单个主机上运行许多服务的系统管理员，手动使用`docker logs`命令逐个容器跟踪和捕获日志可能会在操作上感到疲惫。

在这个技术中，我们将介绍 Docker 的日志驱动功能。这让你可以使用标准的日志系统来跟踪单个主机上的多个服务，甚至跨多个主机。

**问题**

你想在 Docker 主机上集中捕获`docker logs`输出。

**解决方案**

使用`--log-driver`标志将日志重定向到所需的位置。

默认情况下，Docker 日志被捕获在 Docker 守护进程中，您可以使用`docker logs`命令访问这些日志。如您所知，这显示了容器主进程的输出。

在撰写本文时，Docker 提供了多个选项来将此输出重定向到多个`log drivers`，包括

+   syslog

+   journald

+   json-file

默认为 json-file，但可以使用`--log-driver`标志选择其他选项。syslog 和 journald 选项将日志输出发送到同名守护进程。您可以在[`docs.docker.com/engine/reference/logging/`](https://docs.docker.com/engine/reference/logging/)找到所有可用日志驱动程序的官方文档。


##### 警告

此技术需要 Docker 版本 1.6.1 或更高版本。


syslog 守护进程是在服务器上运行的过程，它收集和管理发送到中央文件（通常是 Unix 域套接字）的消息。它通常使用`/dev/log`作为接收日志消息的文件，并将日志输出到`/var/log/syslog`。

Journald 是一个系统服务，用于收集和存储日志数据。它创建并维护一个结构化索引，记录来自各种来源的日志。可以使用`journalctl`命令查询日志。

**将日志记录到 syslog**

要将输出定向到 syslog，请使用`--log-driver`标志：

```
$ docker run --log-driver=syslog ubuntu echo 'outputting to syslog'
outputting to syslog
```

这将记录 syslog 文件中的输出。如果您有权限访问该文件，可以使用标准 Unix 工具检查日志：

```
$ grep 'outputting to syslog' /var/log/syslog
Jun 23 20:37:50 myhost docker/6239418882b6[2559]: outputting to syslog
```

**将日志记录到 journald**

输出到日志守护进程看起来类似：

```
$ docker run --log-driver=journald ubuntu echo 'outputting to journald'
outputting to journald
$ journalctl | grep 'outputting to journald'
Jun 23 11:49:23 myhost docker[2993]: outputting to journald
```


##### 警告

在运行前面的命令之前，请确保您的宿主机上运行着日志守护进程。


**应用于所有容器**

将此参数应用于您主机上的所有容器可能很费力，因此您可以将 Docker 守护进程更改为默认使用这些支持的机制进行日志记录。

修改守护进程`/etc/default/docker`、`/etc/sysconfig/docker`或您的发行版设置的任何 Docker 配置文件，以便激活`DOCKER_OPTS=""`行并包含日志驱动程序标志。例如，如果该行是

```
DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4"
```

更改为：

```
DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4 --log-driver syslog"
```


##### 小贴士

有关如何在您的宿主机上更改 Docker 守护进程配置的详细信息，请参阅附录 B。


如果您重新启动 Docker 守护进程，容器应将日志记录到相关服务。

**讨论**

在此上下文中值得提及的另一个常见选择（但在此处未涵盖）是，您可以使用容器来实现 ELK（Elasticsearch、Logstash、Kibana）日志基础设施。


##### 警告

将此守护进程设置更改为`json-file`或`journald`以外的任何内容，将意味着默认情况下标准`docker logs`命令将不再工作。此 Docker 守护进程的用户可能不会欣赏这种变化，尤其是因为`/var/log/syslog`文件（由`syslog`驱动程序使用）通常对非 root 用户不可访问。

|  |
|  |

**使用 cAdvisor 监控容器**

一旦在生产环境中运行了大量容器，你将希望像在主机上运行多个进程时那样，精确地监控它们的资源使用情况和性能。

监控领域（无论是普遍的，还是针对 Docker）是一个广泛的领域，有许多候选者。在这里选择了 cAdvisor，因为它是一个流行的选择。由 Google 开源，它迅速获得了人气。如果你已经使用 Zabbix 或 Sysdig 等传统主机监控工具，那么值得看看它是否已经提供了你需要的功能——许多工具正在添加容器感知功能，正如我们编写时一样。

**问题**

你想监控你容器的性能。

**解决方案**

使用 cAdvisor 作为监控工具。

cAdvisor 是由 Google 开发的一款用于监控容器的工具。它在 GitHub 上开源，网址为 [`github.com/google/cadvisor`](https://github.com/google/cadvisor)。

cAdvisor 以守护进程的形式运行，收集正在运行的容器的性能数据。其中之一，它跟踪

+   资源隔离参数

+   历史资源使用情况

+   网络统计信息

cAdvisor 可以在主机上本地安装或作为 Docker 容器运行。

##### 列表 15.3\. 运行 cAdvisor

```
$ docker run \                                     *1*
--volume /:/rootfs:ro \                            *2*
 --volume /var/run:/var/run:rw \
 --volume /sys:/sys:ro \                           *3*
 --volume /var/lib/docker/:/var/lib/docker:ro \    *4*
 -p 8080:8080 -d --name cadvisor \                 *5*
 --restart on-failure:10 google/cadvisor           *6*
```

+   ***1*** **允许 cAdvisor 以只读方式访问根文件系统，以便它可以跟踪有关主机的信息**

+   ***2*** **以读写访问权限挂载 /var/run 文件夹。预计每个主机上最多运行一个 cAdvisor 实例。**

+   ***3*** **允许 cAdvisor 以只读方式访问主机的 /sys 文件夹，其中包含有关内核子系统和连接到主机的设备的信息**

+   ***4*** **允许 cAdvisor 以只读方式访问 Docker 的主机目录**

+   ***5*** **cAdvisor 的 Web 界面在容器的 8080 端口上提供服务，因此我们在同一端口上将其发布到主机。运行容器的标准 Docker 参数也被用于在后台运行容器并给容器命名。**

+   ***6*** **在失败时重启容器，最多重启 10 次。镜像存储在 Docker Hub 上，属于 Google 的账户。**

一旦启动了镜像，你可以使用浏览器访问 http://localhost:8080 来开始检查数据输出。这里有关于主机的信息，但通过点击主页顶部的 Docker 容器链接，你可以检查 CPU、内存和其他历史数据的图表。只需点击“子容器”标题下列出的运行容器即可。

在容器运行期间，数据被收集并保存在内存中。GitHub 页面上有关于将数据持久化到 InfluxDB 实例的文档。GitHub 仓库还提供了关于 REST API 和用 Go 编写的示例客户端的详细信息。


##### 提示

InfluxDB 是一个开源数据库，旨在处理时间序列数据的跟踪。因此，它非常适合记录和分析实时提供的监控信息。


**讨论**

监控是一个快速发展和分化的领域，cAdvisor 只是众多组件之一。例如，Prometheus，作为 Docker 的快速崛起的标准，可以接收和存储由 cAdvisor 产生而不是直接放入 InfluxDB 的数据。

监控也是开发者可能非常热衷的一个主题。制定一个灵活的监控策略，以适应不断变化的潮流是有益的。

| |
| --- |

### 15.2. 资源控制

运行生产环境中的服务时，一个核心的担忧是资源的公平和有效分配。在底层，Docker 使用核心操作系统概念 cgroups 来管理容器的资源使用。默认情况下，当容器争夺资源时，使用的是简单且均等份额的算法，但有时这还不够。您可能希望出于运营或服务原因，为容器或容器类别预留或限制资源。

在本节中，您将学习如何调整容器的 CPU 和内存使用。

| |
| --- |

**限制容器可以执行的 CPU 核心**

默认情况下，Docker 允许容器在您的机器上的任何核心上执行。具有单个进程和线程的容器显然只能使用一个核心，但容器中的多线程程序（或多个单线程程序）将能够使用所有 CPU 核心。如果您有一个比其他容器更重要的容器，您可能想要改变这种行为——对于面向客户的应用程序来说，每次内部日常报告运行时都要争夺 CPU 并不理想。您还可以使用这种技术来防止失控的容器阻止您通过 SSH 访问服务器。

**问题**

你希望容器拥有最低的 CPU 分配，对 CPU 消耗有硬性限制，或者想要限制容器可以运行的 CPU 核心数。

**解决方案**

使用`--cpuset-cpus`选项为您的容器预留 CPU 核心。

要正确探索`--cpuset-cpus`选项，您需要在具有多个核心的计算机上执行此技术。如果您使用的是云机器，可能不是这种情况。

| |
| --- |

##### **提示**

旧版本的 Docker 使用`--cpuset`标志，现在已弃用。如果您无法使`--cpuset-cpus`工作，请尝试使用`--cpuset`代替。

| |
| --- |

要查看`--cpuset-cpus`选项的效果，我们将使用`htop`命令，它提供了计算机核心使用情况的直观图形视图。在继续之前，请确保已安装此命令——它通常作为系统包管理器中的`htop`包提供。或者，您可以在使用`--pid=host`选项启动的 Ubuntu 容器内安装它，以便将主机进程信息暴露给容器。

如果您现在运行`htop`，您可能会看到没有任何核心在使用。为了在几个容器内模拟一些负载，请在两个不同的终端中运行以下命令：

```
docker run ubuntu:14.04 sh -c 'cat /dev/zero >/dev/null'
```

回顾`htop`，你应该会看到现在有两个核心显示 100%的使用率。要将其限制在一个核心上，使用`docker kill`终止之前的容器，然后在两个终端中运行以下命令：

```
docker run --cpuset-cpus=0 ubuntu:14.04 sh -c 'cat /dev/zero >/dev/null'
```

现在`htop`将显示这些容器只使用了你的第一个核心。

`--cpuset-cpus`选项允许以逗号分隔的列表（`0,1,2`）、范围（`0-2`）或两者的组合（`0-1,3`）指定多个核心。因此，为宿主机保留 CPU 是一个选择范围的问题，这个范围不包括任何核心。

**讨论**

你可以用多种方式使用这个功能。例如，你可以通过持续将剩余的 CPU 分配给运行中的容器来为宿主进程保留特定的 CPU。或者，你也可以将特定的容器限制在它们自己的专用 CPU 上运行，这样它们就不会干扰其他容器的计算。

在多租户环境中，这可以确保工作负载不会相互干扰，真是一个天赐之物。

|  |

**为重要容器分配更多 CPU**

当容器在宿主机上竞争 CPU 时，它们通常会平均共享 CPU 使用率。你已经看到了如何做出绝对的保证或限制，但这些可能有点不灵活。如果你想让一个进程能够比其他进程使用更多的 CPU，那么不断为它保留整个核心是浪费的，如果你有很少的核心，这样做可能会有限制。

Docker 为希望将应用程序带到共享服务器的用户提供了多租户支持。这可能导致那些有虚拟机经验的人所熟知的*嘈杂邻居*问题，其中一个用户消耗了资源并影响了另一个用户在相同硬件上运行的虚拟机。

作为具体例子，当我们编写这本书时，我们必须使用这个功能来减少一个特别贪婪的 Postgres 应用程序的资源使用，该应用程序消耗了 CPU 周期，剥夺了机器上 Web 服务器为最终用户提供服务的能力。

**问题**

你希望能够给更重要的一些容器分配更多的 CPU 份额，或者将某些容器标记为不太重要。

**解决方案**

使用`docker run`命令的`-c/--cpu-shares`参数来定义 CPU 使用的相对份额。

当容器启动时，它会被分配一个数字（默认为 1024）的*CPU 份额*。当只有一个进程运行时，如果需要，它将能够访问 100%的 CPU，无论它有多少 CPU 份额。只有在与其他容器竞争 CPU 时，这个数字才会被使用。

假设我们有三个容器（A、B 和 C）都在尝试使用所有可用的 CPU 资源：

+   如果它们都被分配了相同的 CPU 份额，那么每个容器将分配到 CPU 的三分之一。

+   如果 A 和 B 被分配了 512，C 被分配了 1024，那么 C 将获得一半的 CPU，A 和 B 各自获得四分之一。

+   如果 A 被分配 10，B 被分配 100，C 被分配 1000，A 将只能获得可用 CPU 资源的不到 1%，并且只有在 B 和 C 空闲时才能执行资源密集型操作。

所有这些都假设你的容器可以使用你机器上的所有核心（或者你只有一个核心）。Docker 将在可能的情况下将容器的负载分散到所有核心。如果你在一个双核心机器上运行两个单线程应用程序，显然没有方法可以在最大化使用可用资源的同时应用相对权重。每个容器都将被分配一个核心来执行，而不管它的权重如何。

如果你想尝试一下，运行以下命令：

##### 列表 15.4\. 使 Docker shell 缺乏 CPU

```
docker run --cpuset-cpus=0 -c 10000 ubuntu:14.04 \
sh -c 'cat /dev/zero > /dev/null' &
docker run --cpuset-cpus=0 -c 1 -it ubuntu:14.04 bash
```

现在看看在 bash 提示符下做任何事情都多么缓慢。请注意，这些数字是相对的——你可以将它们都乘以 10（例如），它们将意味着完全相同的事情。但是默认的授予值仍然是 1024，所以一旦你开始更改这些数字，考虑一下没有在命令中指定 CPU 份额且在相同 CPU 集上运行的过程会发生什么就很有价值了。

| |
| --- |

##### 小贴士

为您的用例找到正确的 CPU 份额级别是一种艺术。查看 top 和 vmstat 等程序的输出以确定什么在消耗 CPU 时间很有价值。当使用 top 时，特别有用的是按“1”键来显示每个 CPU 核心分别在做些什么。

| |
| --- |

**讨论**

尽管我们在现实世界中很少直接看到这种技术的使用，而且它的使用通常在底层平台上看到，但了解并玩转底层机制以了解当租户抱怨缺乏（或明显的缺乏）资源访问时它是如何工作的，这很好。这在现实世界环境中很常见，尤其是如果租户的工作负载对基础设施可用性的波动敏感。

| |
| --- |
| |

**限制容器内存使用**

当你运行一个容器时，Docker 将允许它从主机分配尽可能多的内存。通常这是可取的（并且与虚拟机相比有一个很大的优势，虚拟机有固定的内存分配方式）。但有时应用程序可能会失控，分配过多的内存，并在开始交换时使机器缓慢下来。这很烦人，我们过去曾多次遇到过这种情况。我们想要一种限制容器内存消耗的方法来防止这种情况。

**问题**

你想要能够限制容器的内存消耗。

**解决方案**

使用`docker run`的`-m/--memory`参数。

如果你正在运行 Ubuntu，那么你很可能默认没有启用内存限制功能。要检查，请运行 `docker info`。如果输出中的某一行是关于 `No swap limit support` 的警告，那么不幸的是，你需要做一些设置工作。请注意，进行这些更改可能会对你的机器上所有应用程序的性能产生影响——有关更多信息，请参阅 Ubuntu 安装文档 ([`docs.docker.com/engine/installation/ubuntulinux/#adjust-memory-and-swap-accounting`](http://docs.docker.com/engine/installation/ubuntulinux/#adjust-memory-and-swap-accounting))。

简而言之，你需要在启动时向内核指示你想要这些限制可用。为此，你需要按照以下方式修改 /etc/default/grub。如果 `GRUB_CMDLINE_LINUX` 已经有值，请将新值添加到末尾：

```
-GRUB_CMDLINE_LINUX=""
+GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
```

现在，你需要运行 `sudo update-grub` 并重新启动你的计算机。运行 `docker info` 应该不再显示警告，你现在可以继续进行主要活动了。

首先，让我们通过使用最低可能的限制 4 MB 来粗略地演示内存限制确实起作用。

##### 列表 15.5\. 为容器设置最低可能的内存限制

```
$ docker run -it -m 4m ubuntu:14.04 bash             *1*
 root@cffc126297e2:/# \
python3 -c 'open("/dev/zero").read(10*1024*1024)'    *2*
 Killed                                              *3*
 root@e9f13cacd42f:/# \
A=$(dd if=/dev/zero bs=1M count=10 | base64)         *4*
 $                                                   *5*
 $ echo $?                                           *6*
 137                                                 *7*
```

+   ***1*** **以 4 MB 的内存限制运行容器**

+   ***2*** **尝试将大约 10 MB 的内容加载到内存中**

+   ***3*** **进程消耗了过多的内存，因此被终止。**

+   ***4*** **尝试将 10 MB 的内存直接加载到 bash 中**

+   ***5*** **Bash 被终止，因此容器退出了。**

+   ***6*** **检查退出代码**

+   ***7*** **退出代码非零，表示容器因错误而退出。**

这种类型的约束有一个需要注意的地方。为了演示这一点，我们将使用 jess/stress 镜像，它包含 `stress` 工具，这是一个用于测试系统极限的工具。


##### 小贴士

Jess/stress 是一个用于测试你对容器施加的任何资源限制的有用镜像。如果你想进行更多实验，请尝试使用此镜像之前的技巧。


如果你运行以下命令，你可能会惊讶地发现它并没有立即退出：

```
docker run -m 100m jess/stress --vm 1 --vm-bytes 150M --vm-hang 0
```

你已要求 Docker 将容器限制在 100 MB，并且你已指示 `stress` 使用 150 MB。你可以通过运行此命令来验证 `stress` 是否按预期运行：

```
docker top <container_id> -eo pid,size,args
```

大小列以 KB 为单位，显示你的容器确实使用了大约 150 MB 的内存，这引发了为什么它没有被终止的问题。实际上，Docker 对内存进行了双重预留——一半用于物理内存，另一半用于交换空间。如果你尝试以下命令，容器将立即终止：

```
docker run -m 100m jess/stress --vm 1 --vm-bytes 250M --vm-hang 0
```

这种双重预留只是一个默认设置，可以通过 `--memory-swap` 参数来控制，该参数指定了总虚拟内存大小（内存 + 交换空间）。例如，要完全消除交换空间的使用，应将 `--memory` 和 `--memory-swap` 设置为相同的大小。您可以在 Docker `run` 参考文档中查看更多示例：[`docs.docker.com/engine/reference/run/#user-memory-constraints`](https://docs.docker.com/engine/reference/run/#user-memory-constraints)。

**讨论**

内存限制是任何运行 Docker 平台的运营（或 DevOps）团队的热门话题之一。配置不当或配置不佳的容器经常耗尽分配（或预留）的内存（我看着你，Java 开发者！），需要编写常见问题解答和操作手册来指导用户在遇到问题时如何操作。

了解这里发生的事情对于支持此类平台并向用户提供正在发生的事情的背景非常有帮助。


### 15.3. Docker 的系统管理员用例

在本节中，我们将探讨 Docker 可以用于的一些令人惊讶的用途。尽管乍一看可能觉得奇怪，但 Docker 可以使您的计划任务管理更加容易，并且可以用作备份工具的一种形式。


##### 提示

计划任务（cron job）是一种定时、定期的命令，由几乎所有 Linux 系统中作为服务包含的守护进程运行。每个用户都可以指定自己的命令运行计划。系统管理员广泛使用它来运行周期性任务，例如清理日志文件或运行备份。


这绝不是潜在用途的详尽列表，但它应该让您尝到 Docker 的灵活性，并对其功能如何以意想不到的方式使用有一些了解。


**使用 Docker 运行计划任务**

如果您曾经需要在多个主机上管理计划任务，您可能已经遇到过必须将相同的软件部署到多个位置并确保 crontab 本身具有您想要运行的程序的正确调用的操作难题。

尽管有其他解决方案可以解决这个问题（例如使用 Chef、Puppet、Ansible 或其他配置管理工具来管理跨主机的软件部署），但一个选项是使用 Docker 仓库来存储正确的调用。

这并不是解决上述问题的最佳方案，但它是一个引人注目的例子，说明了拥有一个隔离和可移植的应用程序运行时配置存储库的好处，如果您已经使用 Docker，这将免费获得。

**问题**

您希望您的计划任务能够集中管理和自动更新。

**解决方案**

将您的计划任务脚本作为 Docker 容器拉取并运行。

如果您有一大批需要定期运行作业的机器，您通常会使用 crontabs 并手动配置它们（是的，这仍然会发生），或者您会使用 Puppet 或 Chef 等配置管理工具。更新它们的食谱将确保当机器的配置管理控制器下次运行时，更改将应用到 crontab 中，以便在之后的运行中执行。


##### 小贴士

一个 *crontab* 文件是由用户维护的特定文件，它指定了脚本应该运行的时间。通常这些会是维护任务，比如压缩和存档日志文件，但它们也可能是业务关键应用程序，例如信用卡支付结算器。


在这个技术中，我们将向您展示如何使用来自注册表的 Docker 镜像替换此方案，并通过 'docker pull' 来交付。

在正常情况下，如图 15.2 所示，维护者更新配置管理工具，然后在代理运行时将其交付到服务器。同时，cron 作业在旧代码和新代码之间运行，而系统在更新。

##### 图 15.2\. 每个服务器在 CM 代理计划运行期间更新 cron 脚本

![](img/15fig02.jpg)

在 Docker 场景中，如图 15.3 所示，服务器在 cron 作业运行之前会拉取代码的最新版本。

##### 图 15.3\. 每个服务器在每次 cron 作业运行时都会拉取最新镜像

![](img/15fig03.jpg)

在这一点上，您可能想知道，如果您已经有了可行的解决方案，为什么还要费心去做这件事。以下是使用 Docker 作为交付机制的一些优点：

+   每次运行一个作业时，作业将从中央位置更新到最新版本。

+   您的 crontab 文件变得更加简单，因为脚本和代码都封装在 Docker 镜像中。

+   对于更大或更复杂的更改，只需要拉取 Docker 镜像的增量，从而加快交付和更新速度。

+   您不需要在机器本身上维护代码或二进制文件。

+   您可以将 Docker 与其他技术结合使用，例如将日志输出到 syslog，以简化并集中管理这些管理服务。

在这个例子中，我们将使用我们在 技术 49 中创建的 log_cleaner 镜像。您无疑会记得这个镜像封装了一个清理服务器上日志文件的脚本，并接受一个参数来指定要清理的日志文件的天数。使用 Docker 作为交付机制的 crontab 将类似于以下列表。

##### 列表 15.6\. 日志清理器 crontab 条目

```
0 0 * * * \                                            *1*
 IMG=dockerinpractice/log_cleaner && \
docker pull $IMG && \                                  *2*
 docker run -v /var/log/myapplogs:/log_dir $IMG 1      *3*
```

+   ***1*** **每天午夜运行此操作**

+   ***2*** **首先拉取镜像的最新版本**

+   ***3*** **运行日志清理器处理一天的日志文件**


##### 小贴士

如果你不太熟悉 cron，你可能想知道要编辑你的 crontab，你可以运行`crontab -e`。每一行指定一行开始处的五个项目所指定的时间运行的命令。通过查看 crontab man 页面了解更多信息。

| |
| --- |

如果出现故障，标准的 cron 发送电子邮件机制应该启动。如果你不依赖这个，添加一个带有`or`操作符的命令。在以下示例中，我们假设你的定制警报命令是`my_alert_command`。

##### 列表 15.7\. 在错误时带有警报的日志清理 crontab 条目

```
0 0 * * * \
(IMG=dockerinpractice/log_cleaner && \
docker pull $IMG && \
docker run -v /var/log/myapplogs:/log_dir $IMG 1) \
|| my_alert_command 'log_cleaner failed'
```

| |
| --- |

##### 提示

一个`or`操作符（在这种情况下，双竖线：`||`）确保运行任一侧的命令。如果第一个命令失败（在这种情况下，cron 指定`0 0 * * *`后面的括号内的两个命令之一，由`and`操作符`&&`连接），则将运行第二个命令。

| |
| --- |

`||`操作符确保如果日志清理作业的任何部分运行失败，则将运行警报命令。

**讨论**

我们真的很喜欢这种技术，因为它简单，并且使用经过实战检验的技术以独特的方式解决问题。

Cron 已经存在了几十年（根据维基百科，自 1970 年代末以来）并且通过 Docker 镜像的增强是我们在家以简单方式管理常规任务所使用的技术。

| |
| --- |
| |

**备份的“保存游戏”方法**

如果你曾经运行过事务性系统，你会知道当事情出错时，推断出问题发生时系统状态的能力对于根本原因分析是至关重要的。

通常这是通过多种手段的组合来完成的：

+   应用程序日志分析

+   数据库取证（确定在特定时间点数据的状态）

+   构建历史分析（确定在特定时间点上服务上运行了哪些代码和配置）

+   实时系统分析（例如，是否有人登录到该设备并更改了某些内容？）

对于如此关键的系统，采取简单但有效的方法备份 Docker 服务容器可能是有益的。尽管你的数据库可能与你 Docker 基础设施分开，但配置、代码和日志的状态可以通过几个简单的命令存储在注册表中。

**问题**

你希望保留 Docker 容器的备份。

**解决方案**

在运行时提交容器，并将生成的镜像推送到专门的 Docker 仓库。

遵循 Docker 最佳实践并利用一些 Docker 功能可以帮助你避免需要存储容器备份的需求。例如，使用技术 102 中描述的日志驱动程序而不是将日志记录到容器文件系统中，这意味着不需要从容器备份中检索日志。

但有时现实迫使你不能按照你希望的方式做所有事情，你真的需要看到容器的外观。以下命令显示了提交和推送备份容器的整个过程。

##### 列表 15.8\. 提交和推送备份容器

```
DATE=$(date +%Y%m%d_%H%M%S)                                                *1*
 TAG="your_log_registry:5000/live_pmt_svr_backup:$(hostname -s)_${DATE}"   *2*
 docker commit -m="$DATE" -a="Backup Admin" live_pmt_svr $TAG              *3*
 docker push $TAG                                                          *4*
```

+   ***1*** **生成一个精确到秒的时间戳**

+   ***2*** **生成一个指向您的注册表 URL 的标签，该标签包含主机名和日期**

+   ***3*** **以日期作为消息，并以“备份管理员”作为作者提交容器**

+   ***4*** **将容器推送到注册表**


##### 警告

此技术将在容器运行时暂停它，有效地将其从服务中移除。您的服务应该能够容忍中断，或者您应该有其他节点在此时运行，以负载均衡的方式处理请求。


如果在所有主机上以交错轮换的方式执行此操作，您将拥有一个有效的备份系统，以及尽可能减少歧义地恢复支持工程师状态的手段。图 15.4 展示了这种设置的简化视图。

##### 图 15.4\. 两个主机备份的服务

![](img/15fig04_alt.jpg)

备份仅推送基本镜像和备份时容器状态的差异，并且备份是交错进行的，以确保至少在一个主机上服务保持运行。注册服务器只存储每个提交点的基镜像和差异，节省磁盘空间。

**讨论**

您可以通过结合所谓的“凤凰部署”模型将此技术进一步发展。凤凰部署是一种强调尽可能多地替换系统而不是就地升级部署的部署模型。这是许多 Docker 工具的核心原则。

在这种情况下，而不是提交容器并在之后继续，您可以执行以下操作：

1.  从您的注册表中拉取最新镜像的副本

1.  停止正在运行的容器

1.  启动一个新的容器

1.  提交、标记并推送旧容器到注册表

结合这些方法可以提供更多确定性，即实时系统没有从源镜像中漂移。我们中的一人使用这种方法来管理家庭服务器上的实时系统。


### 摘要

+   您可以将容器的日志直接定向到主机上的 syslog 守护进程。

+   Docker 日志输出可以捕获到主机级别的服务。

+   cAdvisor 可用于监控容器性能。

+   容器对 CPU、核心和内存的使用可以受到限制和控制。

+   Docker 有一些令人惊讶的使用方式，例如作为 cron 交付工具和备份系统。

## 第十六章。在生产中处理 Docker 的挑战


**本章涵盖**

+   绕过 Docker 的命名空间功能并直接使用宿主机的资源

+   确保主机操作系统不会因为内存不足而杀死容器中的进程

+   直接使用宿主机的工具调试容器的网络

+   跟踪系统调用以确定为什么容器在您的宿主机上无法工作


在本章中，我们将讨论当 Docker 的抽象不适合你时你可以做什么。这些主题必然涉及到深入了解 Docker，以了解为什么需要这样的解决方案，在这个过程中，我们旨在让你对使用 Docker 时可能出错的情况有更深入的了解，以及如何着手修复这些问题。

### 16.1. 性能：你不能忽视细节

尽管 Docker 试图将应用程序从其运行的主机抽象出来，但人们永远不能完全忽视主机。为了提供其抽象，Docker 必须添加间接层。这些层可能对你的运行系统有影响，有时为了解决操作挑战或绕过它们，需要理解这些层。

在本节中，我们将探讨如何绕过这些抽象之一，最终得到一个几乎不含 Docker 剩余部分的 Docker 容器。我们还将展示，尽管 Docker 似乎抽象掉了你使用的存储细节，但有时这可能会对你造成伤害。


**从容器访问主机资源**

我们在技术 34 中介绍了最常用的 Docker 抽象绕过方法——卷。它们便于从主机共享文件，并防止大文件进入镜像层。与容器文件系统相比，它们在文件系统访问方面也可能显著更快，因为某些存储后端对某些工作负载施加了显著的开销——这并不是所有应用程序都需要的，但在某些情况下是至关重要的。

除了某些存储后端强加的开销之外，由于 Docker 设置的用于给每个容器提供其自己的网络的网络接口，还会产生另一个性能损失。与文件系统性能一样，网络性能绝对不是每个人的瓶颈，但可能是你希望自行基准测试的东西（尽管网络调优的详细内容远远超出了本书的范围）。或者，你可能有自己的原因想要完全绕过 Docker 网络——一个打开随机端口以监听的服务器可能不会在 Docker 上监听端口范围得到很好的服务，特别是由于暴露的端口范围将在主机上分配，无论它们是否在使用中。

无论你的原因是什么，有时 Docker 的抽象会阻碍你的工作，而 Docker 确实提供了退出选项，如果你需要的话。

**问题**

你希望允许从容器访问主机的资源。

**解决方案**

使用 Docker 提供的`docker run`标志来绕过 Docker 使用的内核命名空间功能。


##### **提示**

内核命名空间是内核提供给程序的服务，允许程序以这种方式获取全局资源的视图，即它们似乎有自己的独立实例。例如，一个程序可以请求一个网络命名空间，这将给你一个看似完整的网络堆栈。Docker 使用并管理这些命名空间来创建其容器。


表 16.1 总结了 Docker 如何使用命名空间，以及如何有效地关闭它们。

##### 表 16.1\. 命名空间和 Docker

| **内核命名空间** | **描述** | **在 Docker 中使用？** | “**关闭选项**” |
| --- | --- | --- | --- |
| 网络 | 网络子系统 | 是 | --net=host |
| IPC | 进程间通信：共享内存、信号量等 | 是 | --ipc=host |
| UTS | 主机名和 NIS 域 | 是 | --uts=host |
| PID | 进程 ID | 是 | --pid=host |
| 挂载 | 挂载点 | 是 | --volume, --device |
| 用户 | 用户和组 ID | 否 | N/A |
|  |

##### **注意**

如果这些标志中的任何一个不可用，那很可能是由于你的 Docker 版本过旧。


如果你的应用程序是共享内存的重度使用者，例如，并且你希望容器与宿主机共享这个空间，你可以使用`--ipc=host`标志来实现这一点。这种用法相对高级，所以我们将会关注其他更常见的用法。

**网络和主机名**

要使用宿主机的网络，你需要使用`--net`标志将容器设置为`host`，如下所示：

```
user@yourhostname:/$ docker run -ti --net=host ubuntu /bin/bash
root@yourhostname:/#
```

你会注意到，这与网络命名空间容器立即不同，因为容器内的主机名与宿主机相同。在实用层面上，这可能会导致混淆，因为不明显知道自己在容器中。

在一个网络隔离的容器中，快速执行`netstat`命令将显示启动时没有连接：

```
host$ docker run -ti ubuntu
root@b1c4877a00cd:/# netstat
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path
root@b1c4877a00cd:/#
```

使用宿主机的网络运行类似操作将显示一个忙碌的技术作者通常的网络繁忙的主机：

```
$ docker run -ti --net=host ubuntu
root@host:/# netstat -nap | head
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address   Foreign Address State       PID
 /Program name
tcp        0      0 127.0.0.1:47116 0.0.0.0:*       LISTEN      -
tcp        0      0 127.0.1.1:53    0.0.0.0:*       LISTEN      -
tcp        0      0 127.0.0.1:631   0.0.0.0:*       LISTEN      -
tcp        0      0 0.0.0.0:3000    0.0.0.0:*       LISTEN      -
tcp        0      0 127.0.0.1:54366 0.0.0.0:*       LISTEN      -
tcp        0      0 127.0.0.1:32888 127.0.0.1:47116 ESTABLISHED -
tcp        0      0 127.0.0.1:32889 127.0.0.1:47116 ESTABLISHED -
tcp        0      0 127.0.0.1:47116 127.0.0.1:32888 ESTABLISHED -
root@host:/#
```


##### **注意**

`netstat`是一个命令，允许你查看本地网络堆栈上的网络信息。它最常用于确定网络套接字的状态。


使用`net=host`标志的原因有很多。首先，它可以使连接容器变得更容易。但你会失去容器端口映射的好处。例如，如果你有两个监听 80 端口的容器，你不能以这种方式在同一个主机上运行它们。第二个原因是，使用此标志时，网络性能比 Docker 本身有显著提升。

图 16.1 从高层次展示了在 Docker 与原生网络中，网络数据包必须经过的额外层。原生网络只需要通过主机的 TCP/IP 堆栈到网络接口卡（NIC），而 Docker 还需要维护一个虚拟以太对（一个“veth pair”——通过以太网电缆的物理连接的虚拟表示），一个在此 veth 对和主机网络之间的网络桥，以及一层网络地址转换（NAT）。这种开销可能导致 Docker 网络在正常使用情况下速度仅为原生主机网络的一半。

##### 图 16.1\. Docker 网络与原生网络

![](img/16fig01.jpg)

**PID**

PID 命名空间标志与其他标志类似：

```
imiell@host:/$ docker run ubuntu ps -p 1                    *1*
   PID TTY          TIME CMD
    1 ?        00:00:00 ps                                  *2*
 imiell@host:/$ docker run --pid=host ubuntu ps -p 1        *3*
   PID TTY          TIME CMD
    1 ?        00:00:27 systemd                             *4*
```

+   ***1*** **在容器化环境中运行 ps 命令，只显示具有 PID 1 的进程**

+   ***2*** **我们运行的 ps 是这个容器中唯一的进程，并赋予它 PID 1。**

+   ***3*** **运行不带 PID 命名空间的相同 ps 命令，从而让我们看到主机进程**

+   ***4*** **这次 PID 1 是 systemd 命令，它是主机操作系统的启动进程。这可能会因你的发行版而异。**

上述示例演示了主机上的 systemd 进程在具有主机 PID 视图的容器中的进程 ID 为 1，而没有这种视图时，唯一看到的进程就是`ps`命令本身。

**挂载**

如果你想访问主机的设备，使用`--device`标志使用特定设备，或者使用`--volume`标志挂载整个主机的文件系统：

```
docker run -ti --volume /:/host ubuntu /bin/bash
```

上述命令将主机的`/`目录挂载到容器的`/host`目录。你可能想知道为什么不能将主机的`/`目录挂载到容器的`/`目录。这是由`docker`命令明确禁止的。

你可能还在想是否可以使用这些标志创建一个几乎与主机无法区分的容器。这引出了下一个部分...

**类似主机的容器**

你可以使用以下标志来创建一个具有几乎透明主机视图的容器：

```
host:/$ docker run -ti --net=host --pid=host --ipc=host \    *1*
 --volume /:/host \                                          *2*
 busybox chroot /host                                        *3*
```

+   ***1*** **运行具有三个主机参数（net、pid、ipc）的容器**

+   ***2*** **将主机根文件系统挂载到容器的`/host`目录。Docker 不允许将卷挂载到“/”文件夹，因此你必须指定`/host`子文件夹卷。**

+   ***3*** **启动一个 BusyBox 容器。你只需要 chroot 命令，这是一个包含该命令的小镜像。Chroot 被执行，使得挂载的文件系统看起来像是你的根目录。**

令人讽刺的是，Docker 被描述为“强化版的`chroot`”，而在这里我们使用的是一个被描述为框架的东西来以颠覆`chroot`主要目的的方式运行`chroot`，即保护主机文件系统。通常在这个时候，我们尽量不去深入思考。

无论如何，很难想象这个命令（尽管有教育意义）在现实世界中有实际用途。如果你想到了一个，请给我们发邮件。

话虽如此，你可能想将其用作以下更有用命令的基础：

```
$ docker run -ti --workdir /host \
   --volume /:/host:ro ubuntu /bin/bash
```

在这个例子中，`--workdir /host`将容器启动时的工作目录设置为宿主文件系统的根目录，正如使用`--volume`参数挂载的那样。卷规范中的`:ro`部分表示宿主文件系统将以只读方式挂载。

使用这个命令，你可以在具有安装工具（使用标准的 Ubuntu 包管理器）以检查文件系统的环境中获得文件系统的只读视图。例如，你可以使用一个运行一个报告主机文件系统安全问题的巧妙工具的镜像，而无需在主机上安装它。

| |
| --- |

##### 警告

如前所述的讨论所暗示的，使用这些标志会使你面临更多的安全风险。在安全术语中，使用它们应被视为与使用`--privileged`标志运行等效。

| |
| --- |

**讨论**

在这个技术中，你已经学会了如何在容器内绕过 Docker 的抽象。禁用这些功能可以给你带来速度提升或其他便利，使 Docker 更好地满足你的需求。我们过去使用的一个变体是在容器内安装网络工具（例如，像在技术 112 中提到的 tcpflow）并暴露主机网络接口。这让你可以临时尝试不同的工具，而无需安装它们。

下一个技术将探讨如何绕过 Docker 底层磁盘存储的限制。

| |
| --- |
| |

**禁用 OOM 杀手**

“OOM 杀手”听起来像一部糟糕的恐怖电影或严重的疾病，但实际上它是 Linux 操作系统内核中的一个线程，当宿主机内存不足时，它决定要做什么。在操作系统耗尽硬件内存、用尽所有可用的交换空间以及从内存中移除所有缓存文件后，它将调用 OOM 杀手来决定哪些进程应该被终止。

**问题**

你想要防止容器被 OOM 杀手杀死。

**解决方案**

在启动容器时使用`--oom-kill-disable`标志。

解决这个挑战就像给你的 Docker 容器添加一个标志一样简单。但正如通常情况那样，整个故事并不那么简单。

下面的列表显示了如何禁用容器的 OOM 杀手：

##### 列表 16.1. `--oom-kill-disable`显示警告

```
$ docker run -ti --oom-kill-disable ubuntu sleep 1                     *1*
 WARNING: Disabling the OOM killer on containers without setting a
 '-m/--memory' limit may be dangerous.                              *2*
```

+   ***1*** **将--oom-kill-disable 标志添加到正常的 docker run 命令中。**

+   ***2*** **输出有关可能设置的另一个标志的警告。**

你看到的警告很重要。它告诉你使用此设置是危险的，但它没有告诉你原因。设置此选项是危险的，因为如果宿主机的内存耗尽，操作系统将在你的进程之前杀死所有其他用户进程。

有时这是可取的，例如，如果您有一项关键的基础设施需要保护免受故障的影响——可能是一个跨（或为）主机上所有容器运行的审计或日志记录过程。即使如此，您也可能会想两次考虑这将对您的环境造成多大的干扰。例如，您的容器可能依赖于同一主机上运行的其他基础设施。如果您在 OpenShift 这样的容器平台上运行，即使关键平台进程被杀死，您的容器也能幸存。您可能希望在该容器之前让关键基础设施保持运行。

##### 列表 16.2\. `--oom-kill-disable` 没有警告

```
$ docker run -ti --oom-kill-disable --memory 4M ubuntu sleep 1      *1*
 $                                                                  *2*
```

+   ***1*** **将 --memory 标志添加到正常的 docker run 命令中。**

+   ***2*** **这次，没有看到警告。**


##### 注意

您可以分配的最小内存量是 4M，其中“M”代表兆字节。您也可以按“G”分配千兆字节。


您可能想知道如何判断您的容器是否被 OOM 杀手杀死。这可以通过使用 `docker inspect` 命令轻松完成：

##### 列表 16.3\. 判断您的容器是否被“OOM-killed”

```
$ docker inspect logger | grep OOMKilled
            "OOMKilled": false,
```

此命令输出容器被杀死的详细信息，包括是否由 OOM 杀手将其杀死。

**讨论**

OOM 杀手不需要在容器中设置扩展权限，也不需要您是 root 用户——您只需要访问 `docker` 命令。这是又一个要小心不要在不信任用户拥有 root 权限的情况下给予他们 `docker` 命令访问权限的理由（参见第十四章关于安全）。

这不仅是一个安全风险，也是一个稳定性风险。如果用户可以运行 `docker`，他们可以运行一个逐渐泄漏内存的过程（这在许多生产环境中很常见）。如果没有对内存设置边界，操作系统将在选项耗尽后介入，并首先杀死内存使用量最大的用户进程（这是对 Linux OOM-killer 算法的简化，该算法经过多年的实战检验并不断完善）。然而，如果容器是以禁用 OOM 杀手的方式启动的，它可能会破坏主机上的所有容器，给用户造成更大的破坏和不稳定。

对于更精细的内存管理方法，您可以使用 `--oom-score-adj` 标志调整容器的“OOM 分数”。另一种可能适合您的方法是在内核中禁用内存过载提交。这将全局关闭 OOM 杀手，因为只有当内存确实可用时才会分配内存。然而，这可能会限制您主机上可以运行的容器数量，这也可能是不希望的。

正如往常一样，性能管理是一门艺术！


### 16.2\. 当容器泄漏时——调试 Docker

在本节中，我们将介绍一些技术，帮助你理解和修复在 Docker 容器中运行的应用程序的问题。我们将介绍如何在使用主机工具调试问题时跳入容器的网络，并查看一个直接监控网络接口以避免容器操作的选择方案。

最后，我们将演示 Docker 抽象如何崩溃，导致容器在一个主机上工作而在另一个主机上不工作，以及如何在实时系统上调试这个问题。


**使用 nsenter 调试容器的网络**

在理想的世界里，你可以在*大使容器*中使用 socat（见技术 4）来诊断容器通信的问题。你会启动额外的容器，并确保连接到这个新的容器，它充当代理。代理允许你诊断和监控连接，然后将它们转发到正确的位置。不幸的是，仅为了调试目的设置这样的容器并不总是方便（或可能）。


##### 提示

有关大使模式的描述，请参阅技术 74。


你已经在技术 15 和 19 中阅读了关于`docker exec`的内容。这项技术讨论了*nsenter*，这是一个看起来相似的工具，但它允许你在容器内部使用来自你机器的工具，而不是仅限于容器安装的工具。

**问题**

你想在容器中调试一个网络问题，但工具不在容器中。

**解决方案**

使用 nsenter 跳入容器的网络，但保留主机的工具。

如果你已经在你的 Docker 主机上安装了 nsenter，你可以使用以下命令构建它：

```
$ docker run -v /usr/local/bin:/target jpetazzo/nsenter
```

这将在/usr/local/bin 中安装 nsenter，你将能够立即使用它。nsenter 也可能在你的发行版（util-linux 包）中可用。

到现在为止，你可能已经注意到，通常有用的 BusyBox 镜像默认不包含 bash。作为 nsenter 的演示，我们将展示如何使用主机的 bash 程序进入 BusyBox 容器：

```
$ docker run -ti busybox /bin/bash
FATA[0000] Error response from daemon: Cannot start container >
a81e7e6b2c030c29565ef7adb94de20ad516a6697deeeb617604e652e979fda6: >
exec: "/bin/bash": stat /bin/bash: no such file or directory
$ CID=$(docker run -d busybox sleep 9999)                      *1*
 $ PID=$(docker inspect --format {{.State.Pid}} $CID)          *2*
 $ sudo nsenter --target $PID \                                *3*
 --uts --ipc --net /bin/bash                                   *4*
 root@781c1fed2b18:~#
```

+   ***1*** **启动 BusyBox 容器并保存容器 ID (CID)**

+   ***2*** **检查容器，提取进程 ID (PID)（见技术 30）**

+   ***3*** **运行 nsenter，使用--target 标志指定要进入的容器。可能不需要“sudo”。**

+   ***4*** **使用剩余的标志指定要进入的容器的命名空间**

有关 nsenter 理解的命名空间的更多详细信息，请参阅技术 109。在命名空间选择的关键点是不要使用`--mount`标志，因为这会使用容器的文件系统，因为 bash 将不可用。/bin/bash 被指定为要启动的可执行文件。

应该指出的是，你无法直接访问容器的文件系统，但你确实拥有主机上的所有工具。

我们之前需要的是一种方法来找出主机上的哪个 veth 接口设备对应于哪个容器。例如，有时快速将容器从网络上移除是有用的。无权限的容器无法关闭网络接口，因此你需要从主机通过找出 veth 接口名称来完成此操作。

```
$ docker run -d --name offlinetest ubuntu:14.04.2 sleep infinity
fad037a77a2fc337b7b12bc484babb2145774fde7718d1b5b53fb7e9dc0ad7b3
$ docker exec offlinetest ping -q -c1 8.8.8.8                              *1*
 PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.966/2.966/2.966/0.000 ms
$ docker exec offlinetest ifconfig eth0 down                               *2*
 SIOCSIFFLAGS: Operation not permitted
$ PID=$(docker inspect --format {{.State.Pid}} offlinetest)
$ nsenter --target $PID --net ethtool -S eth0                              *3*
 NIC statistics:
     peer_ifindex: 53
$ ip addr | grep '⁵³'                                                     *4*
 53: veth2e7d114: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue >
master docker0 state UP
$ sudo ifconfig veth2e7d114 down                                           *5*
 $ docker exec offlinetest ping -q -c1 8.8.8.8                             *6*
 PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

+   ***1*** **验证从新容器内部尝试 ping 是否成功**

+   ***2*** **无法将容器中的接口关闭。请注意，你的接口可能不是 eth0，如果这不起作用，你可能希望使用 ip addr 来找出你的主要接口名称。**

+   ***3*** **进入容器的网络空间，使用主机上的 ethtool 命令查找对端接口索引——虚拟接口的另一端**

+   ***4*** **在主机上的接口列表中查找适合容器的适当 veth 接口**

+   ***5*** **关闭虚拟接口**

+   ***6*** **验证从容器内部尝试 ping 是否失败**

你可能还想在容器内部使用的一个程序示例是 tcpdump，这是一个记录网络接口上所有 TCP 数据包的工具。要使用它，你需要运行带有 `--net` 命令的 `nsenter`，这样你就可以从主机“看到”容器的网络，因此可以使用 tcpdump 监控数据包。

例如，以下代码中的 `tcpdump` 命令将所有数据包记录到 `/tmp/google.tcpdump` 文件中（我们假设你仍然处于之前启动的 nsenter 会话中）。然后通过检索网页来触发一些网络流量：

```
root@781c1fed2b18:/# tcpdump -XXs 0 -w /tmp/google.tcpdump &
root@781c1fed2b18:/# wget google.com
--2015-08-07 15:12:04--  http://google.com/
Resolving google.com (google.com)... 216.58.208.46, 2a00:1450:4009:80d::200e
Connecting to google.com (google.com)|216.58.208.46|:80... connected.
HTTP request sent, awaiting response... 302 Found
Location: http://www.google.co.uk/?gfe_rd=cr&ei=tLzEVcCXN7Lj8wepgarQAQ >
[following]
--2015-08-07 15:12:04--  >
http://www.google.co.uk/?gfe_rd=cr&ei=tLzEVcCXN7Lj8wepgarQAQ
Resolving www.google.co.uk (www.google.co.uk)... 216.58.208.67, >
2a00:1450:4009:80a::2003
Connecting to www.google.co.uk (www.google.co.uk)|216.58.208.67|:80... >
connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: 'index.html'

index.html             [ <=>             ]  18.28K  --.-KB/s   in 0.008s

2015-08-07 15:12:05 (2.18 MB/s) - 'index.html' saved [18720]

root@781c1fed2b18:# 15:12:04.839152 IP 172.17.0.26.52092 > >
google-public-dns-a.google.com.domain: 7950+ A? google.com. (28)
15:12:04.844754 IP 172.17.0.26.52092 > >
google-public-dns-a.google.com.domain: 18121+ AAAA? google.com. (28)
15:12:04.860430 IP google-public-dns-a.google.com.domain > >
172.17.0.26.52092: 7950 1/0/0 A 216.58.208.46 (44)
15:12:04.869571 IP google-public-dns-a.google.com.domain > >
172.17.0.26.52092: 18121 1/0/0 AAAA 2a00:1450:4009:80d::200e (56)
15:12:04.870246 IP 172.17.0.26.47834 > lhr08s07-in-f14.1e100.net.http: >
Flags [S], seq 2242275586, win 29200, options [mss 1460,sackOK,TS val >
49337583 ecr 0,nop,wscale 7], length 0
```


##### 小贴士

根据你的网络设置，你可能需要暂时更改你的 resolv.conf 文件以允许 DNS 查询工作。如果你收到“名称解析临时失败”错误，请尝试将 `nameserver 8.8.8.8` 行添加到你的 /etc/resolv.conf 文件顶部。完成操作后，别忘了恢复。


**讨论**

这种技术为你提供了一种快速更改容器网络行为的方法，而无需使用第十章中提到的任何工具（技术 78 和 79）来模拟网络故障。

你也看到了 Docker 的一个令人信服的使用案例——在 Docker 提供的隔离网络环境中调试网络问题比在不受控制的环境中更容易。在深夜试图记住 tcpdump 的正确参数以适当过滤掉无关数据包是一个容易出错的过程。使用 nsenter，你可以忘记这一点，并捕获容器内的所有内容，而无需在镜像上安装（或必须安装）tcpdump。

|  |

**使用 tcpflow 在不重新配置的情况下进行调试**

tcpdump 是网络调查的事实标准，并且很可能是当被要求深入调试网络问题时，大多数人首先会使用的工具。

但 tcpdump 通常用于显示数据包摘要和检查数据包头部和协议信息——它并不是特别适合显示两个程序之间的应用程序级数据流。当调查两个应用程序通信的问题时，这可能会非常重要。

**问题**

你需要监控容器化应用程序的通信数据。

**解决方案**

使用 tcpflow 捕获跨越接口的流量。

tcpflow 与 tcpdump 类似（接受相同的模式匹配表达式），但它旨在让你更好地了解应用程序数据流。tcpflow 可能可以从你的系统包管理器中获取，如果没有，我们已准备了一个你可以使用的 Docker 镜像，其功能应与等效包管理器安装几乎相同：

```
$ IMG=dockerinpractice/tcpflow
$ docker pull $IMG
$ alias tcpflow="docker run --rm --net host $IMG"
```

你可以使用两种方式与 Docker 一起使用 tcpflow：将其指向 `docker0` 接口，并使用数据包过滤表达式仅检索你想要的包，或者使用前一种技术中的技巧来找到你感兴趣的容器的 veth 接口，并在该接口上捕获。

| |
| --- |

##### 提示

你可能希望参考 第十章 10.2 图 来刷新你对 Docker 内部网络流量流动的记忆，并了解为什么在 `docker0` 上捕获会捕获容器流量。

| |
| --- |

表达式过滤是 tcpflow 在连接到接口后使用的一个强大功能，让你可以深入到你感兴趣的流量。我们将展示一个简单的示例来帮助你入门：

```
$ docker run -d --name tcpflowtest alpine:3.2 sleep 30d
fa95f9763ab56e24b3a8f0d9f86204704b770ffb0fd55d4fd37c59dc1601ed11
$ docker inspect -f '{{ .NetworkSettings.IPAddress }}' tcpflowtest
172.17.0.1
$ tcpflow -c -J -i docker0 'host 172.17.0.1 and port 80'
tcpflow: listening on docker0
```

在前面的示例中，你要求 tcpflow 打印任何流向或来自你的容器且源或目标端口为 80（通常用于 HTTP 流量）的彩色流。你现在可以通过在新终端中检索容器中的网页来尝试这个操作：

```
$ docker exec tcpflowtest wget -O /dev/null http://www.example.com/
Connecting to www.example.com (93.184.216.34:80)
null            100% |*******************************|  1270   0:00:00 ETA
```

你将在 tcpflow 终端中看到彩色输出。到目前为止命令的累积输出将类似于以下内容：

```
$ tcpflow -J -c -i docker0 'host 172.17.0.1 and (src or dst port 80)'
tcpflow: listening on docker0
172.017.000.001.36042-093.184.216.034.00080: >
GET / HTTP/1.1                                   *1*
 Host: www.example.com
User-Agent: Wget
Connection: close

093.184.216.034.00080-172.017.000.001.36042: >
HTTP/1.0 200 OK                                  *2*
 Accept-Ranges: bytes
Cache-Control: max-age=604800
Content-Type: text/html
Date: Mon, 17 Aug 2015 12:22:21 GMT
[...]

<!doctype html>
<html>
<head>
    <title>Example Domain</title>
[...]
```

+   ***1*** **蓝色着色开始**

+   ***2*** **红色着色开始**

**讨论**

tcpflow 是你工具箱的一个很好的补充，因为它非常不引人注目。你可以针对长时间运行的容器启动它，以获得它们现在正在传输的内容的一些洞察，或者与 tcpdump（前一种技术）一起使用，以获得你应用程序发出的请求类型和传输的信息的更完整视图。

除了 tcpdump，前一种技术还涵盖了使用 nsenter 来监控单个容器上的流量，而不是所有容器（这是监控 docker0 会做的事情）。

| |
| --- |
| |

**调试特定主机上失败的容器**

前两种技术已经展示了你可以如何开始调查由你的容器与其他位置（无论是更多容器还是互联网上的第三方）之间的交互引起的问题。

如果你已经将问题隔离到一个主机上，并且确信外部交互不是原因，下一步应该尝试减少可移动部件的数量（移除卷和端口）并检查主机本身的详细信息（可用磁盘空间、打开的文件描述符数量等）。也许还值得检查每个主机是否运行了 Docker 的最新版本。

在某些情况下，上述方法都无济于事——你有一个可以不带参数运行（例如 `docker run imagename`）的镜像，它应该被完美地隔离，但在不同的主机上运行时却有所不同。

**问题**

你想要确定为什么容器内的特定操作在特定主机上不起作用。

**解决方案**

使用 strace 跟踪进程以查看它正在执行哪些系统调用，并将其与正常工作的系统进行比较。

尽管 Docker 声明的目标是让用户“在任何地方运行任何应用”，但它试图实现这一目标的手段并不总是万无一失。

Docker 将 Linux 内核 API 视为其*宿主*（它可以在其中运行的环境）。当人们刚开始学习 Docker 的工作原理时，很多人会问 Docker 如何处理 Linux API 的变化。据我们所知，它还没有这样做。幸运的是，Linux API 是向后兼容的，但可以想象，在未来某个时候，可能会创建一个新的 Linux API 调用，并被 Docker 化的应用程序使用，然后该应用程序被部署到一个足够新以运行 Docker 但旧到不支持该特定 API 调用的内核上。

| |
| --- |

##### 注意

你可能会认为 Linux 内核 API 的变化是一个理论上的问题，但我们在编写这本书的第一版时遇到了这种情况。我们正在开发的一个项目使用了`memfd_create` Linux 系统调用，它只存在于版本 3.17 及以上的内核中。因为一些我们正在工作的主机有较旧的内核，我们的容器在一些系统上失败，而在其他系统上则工作正常。

| |
| --- |

那种场景并不是 Docker 抽象失败的唯一方式。容器可能会因为应用程序对主机上的文件所做的假设而失败。虽然这种情况很少见，但它确实会发生，因此重要的是要警惕这种风险。

**SELinux 对容器的干扰**

Docker 抽象可能崩溃的一个例子是与 SELinux 交互的任何内容。如第十四章讨论所述，SELinux 是在内核中实现的一层安全机制，它工作在正常用户权限之外。

Docker 使用这一层来允许通过管理容器内可以执行的操作来加强容器安全性。例如，如果您在容器内是 root 用户，您与主机上的 root 用户是相同的用户。尽管很难突破容器以获得主机上的 root 权限，但这并非不可能；已经发现了漏洞，可能还有社区尚未知晓的其他漏洞。SELinux 可以做到的是提供另一层保护，即使 root 用户从容器突破到主机，他们可以在主机上执行的操作也有限制。

到目前为止一切顺利，但 Docker 的问题在于 SELinux 是在主机上实现的，而不是在容器内。这意味着在容器中运行的程序查询 SELinux 的状态并发现它已启用时，可能会对其运行的环境做出某些假设，如果这些期望没有得到满足，可能会以意想不到的方式失败。

在以下示例中，我们正在运行一个安装了 Docker 的 CentOS 7 Vagrant 虚拟机，并在其中运行一个 Ubuntu 12.04 容器。如果我们运行一个相当直接的命令来添加用户，退出代码是 12，表示错误，并且确实用户没有被创建：

```
[root@centos vagrant]# docker run -ti ubuntu:12.04
Unable to find image 'ubuntu:12.04' locally
Pulling repository ubuntu
78cef618c77e: Download complete
b5da78899d3a: Download complete
87183ecb6716: Download complete
82ed8e312318: Download complete
root@afade8b94d32:/# useradd -m -d /home/dockerinpractice dockerinpractice
root@afade8b94d32:/# echo $?
12
```

在`ubuntu:14.04`容器上运行的相同命令工作正常。如果您想尝试重现此结果，您需要一个 CentOS 7 机器（或类似）。但为了学习目的，使用任何命令和容器遵循其余的技术将足够。

| |
| --- |

##### 小贴士

在 bash 中，`$?`会给你上一个运行的命令的退出代码。退出代码的含义因命令而异，但通常退出代码为 0 表示调用成功，非零代码表示错误或某种异常情况。

| |
| --- |

**调试 Linux API 调用**

因为我们知道容器之间可能存在的差异是由于主机上运行的内核 API 之间的差异，strace 可以帮助您确定对内核 API 的调用之间的差异。

strace 是一个允许您监视进程（即系统调用）对 Linux API 进行的调用的工具。它是一个极其有用的调试和教育工具。您可以在图 16.2 中看到它是如何工作的。

##### 图 16.2\. strace 的工作原理

![图片](img/16fig02.jpg)

首先，您需要使用适当的包管理器在容器上安装 strace，然后运行不同的命令，并在前面加上`strace`命令。以下是失败的`useradd`调用的示例输出：

```
# strace -f \
                                                                             *1*
 useradd -m -d /home/dockerinpractice dockerinpractice                       *2*
 execve("/usr/sbin/useradd", ["useradd", "-m", "-d", >                       *3*
 "/home/dockerinpractice", "dockerinpractice"], [/* 9 vars */]) = 0
[...]
open("/proc/self/task/39/attr/current", >                                    *4*
 O_RDONLY) = 9
read(9, "system_u:system_r:svirt_lxc_net_"..., >                             *5*
 4095) = 46
close(9)                                = 0                                  *6*
 [...]
open("/etc/selinux/config", O_RDONLY)   = >
                                                                             *7*
 -1 ENOENT (No such file or directory)                                       *7*
open("/etc/selinux/targeted/contexts/files/ >
                                                                             *7*
 file_contexts.subs_dist", O_RDONLY) = -1 ENOENT (No such file or directory) *7*
open("/etc/selinux/targeted/contexts/files/ >
                                                                             *7*
 file_contexts.subs", O_RDONLY) = -1 ENOENT (No such file or directory)
                                                                             *7*
open("/etc/selinux/targeted/contexts/files/ >
                                                                             *7*
 file_contexts", O_RDONLY) = -1 ENOENT (No such file or directory)
                                                                             *7*
[...]
exit_group(12)                                                               *8*
```

+   ***1*** **使用-f 标志运行 strace，这确保了您的命令所产生的过程及其任何后代都会被 strace“跟随”**

+   ***2*** **将您要调试的命令附加到 strace 调用中**

+   ***3*** **strace 输出的每一行都以 Linux API 调用开始。这里的 execve 调用执行您给 strace 的命令。最后的 0 是调用的返回值（成功）。**

+   ***4*** **“open”系统调用打开一个文件以供读取。返回值（9）是后续调用中用于操作文件的文件句柄号。在这种情况下，SELinux 信息是从/proc 文件系统中检索的，该文件系统包含有关运行进程的信息。**

+   ***5*** **“read”系统调用作用于之前打开的文件（文件描述符为 9），并返回读取的字节数（46）。**

+   ***6*** **“close”系统调用关闭了由文件描述符引用的文件。**

+   ***7*** **程序尝试打开它期望存在的 SELinux 文件，但在每种情况下都失败了。strace 有用地告诉你返回值的含义：“没有这样的文件或目录。”**

+   ***8*** **进程以值 12 退出，对于 useradd 来说，这意味着目录无法创建。**

起初，前面的输出可能看起来很令人困惑，但经过几次之后，它就变得相对容易阅读了。每一行代表对 Linux 内核的调用，以在所谓的*内核空间*（与*用户空间*相对，在用户空间中，操作由程序执行，而不将责任交给内核）执行某些操作。


##### **提示**

如果你想了解更多关于特定系统调用的信息，你可以运行`man 2 callname`。你可能需要使用`apt-get install manpages-dev`或类似命令为你的包装系统安装 man 页面。或者，通过 Google 搜索`man 2 callname`可能会得到你需要的信息。


这就是 Docker 的抽象崩溃的例子。在这种情况下，操作失败是因为程序期望 SELinux 文件存在，因为 SELinux 似乎在容器上被启用，但执行细节保留在宿主机上。


##### **提示**

如果你认真对待成为一名开发者，阅读所有系统调用的`man 2`页面非常有用。一开始它们可能看起来充满了你不理解的术语，但随着你对各种主题的阅读，你会学到很多关于 Linux 基本概念的知识。在某个时候，你将开始了解大多数语言是如何从这个根源衍生出来的，它们的一些怪癖和奇怪之处将更有意义。但是要有耐心，因为你不会立即理解所有内容。


**讨论**

虽然这种情况很少见，但使用 strace 进行调试和理解程序如何交互的能力是一种非常有价值的技巧，不仅适用于 Docker，也适用于更广泛的开发生态。

如果你拥有非常小的 Docker 镜像，可能通过利用技术 57 创建，并且你不想在容器上安装 strace，那么你可以使用宿主机的 strace。你需要使用`docker top <container_id>`来找到容器中进程的 PID，并使用 strace 的`-p`参数来附加到特定的运行进程。别忘了使用 sudo。附加到进程可能允许你读取其秘密，因此需要额外的权限。

|  |

**从镜像中提取文件**

使用`docker cp`命令从容器中复制文件很容易实现。不经常地，你可能想要从一个镜像中提取文件，但你没有干净运行的容器来复制。在这些情况下，你可以人为地运行一个镜像的容器，运行`docker cp`，然后删除容器。这已经是三个命令了，如果你，例如，镜像有一个默认的 entrypoint，它需要有效的参数，你可能会遇到麻烦。

这种技术给你一个单一的命令别名，你可以将其放入你的 shell 启动脚本中，通过一个命令和两个参数完成所有这些操作。

**问题**

你想要从镜像复制文件到你的主机。

**解决方案**

使用别名从镜像中运行一个容器，并将 entrypoint 设置为将文件内容输出到主机上的文件。

首先，我们将向你展示如何构建一个`docker run`命令来从镜像中提取文件，然后你会看到如何将其转换为方便的别名。

##### 列表 16.4\. 使用`docker run`从镜像中提取文件

```
$ docker run --rm  \                                *1*
     -i \                                           *2*
     -t \                                           *3*
     --entrypoint=cat \                             *4*
     ubuntu \                                       *5*
     /etc/os-release \                              *6*
     > ubuntu_os-release                            *7*
 $ cat ubuntu_os-release
NAME="Ubuntu"
VERSION="16.04.1 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.1 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
$ cat /etc/os-release                               *8*
 cat: /etc/os-release: No such file or directory
```

+   ***1*** 使用--rm 标志在运行此命令后立即删除容器

+   ***2*** 使用-i 标志使容器交互式

+   ***3*** 使用-t 标志给容器提供一个虚拟终端来写入

+   ***4*** 将容器的 entrypoint 设置为‘cat’

+   ***5*** 你想要从其中提取文件的镜像名称

+   ***6*** 输出的文件名

+   ***7*** 将文件内容重定向到主机上的本地文件

+   ***8*** 为了强调这一点，我们展示了在主机上不存在`/etc/os-release`。

你可能想知道为什么在这里使用`entrypoint`，而不是简单地运行`cat`命令来输出文件。这是因为某些镜像已经设置了一个 entrypoint。当这种情况发生时，Docker 会将`cat`视为`entrypoint`命令的参数，从而导致你不希望的行为。

为了方便，你可能想将此命令放入别名中。

##### 列表 16.5\. 使用别名从镜像中提取文件

```
$ alias imagecat='docker run --rm -i -t --entrypoint=cat'      *1*
 $ imagecat ubuntu /etc/os-release                             *2*
 NAME="Ubuntu"
VERSION="16.04.1 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.1 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
```

+   ***1*** 将命令别名设置为“imagecat”，包含从列表 16.4 到图像和文件参数的所有命令内容

+   ***2*** 使用两个参数（图像和文件名）调用“imagecat”

这种技术假设你的容器中存在`cat`。如果你使用技术 58 构建了最小化容器，这可能不是情况，因为容器中只有你的二进制文件——没有标准 Linux 工具。

如果是这样的话，你可能想考虑使用技术 73 中的`docker export`，但不是将它们发送到另一台机器，而是可以直接从它们中提取你想要的文件。记住，容器不需要成功启动你才能导出它——你可以尝试使用容器内不存在的命令来运行它，然后导出停止的容器（或者直接使用`docker create`，它为执行准备容器而不启动它）。 


### 摘要

+   你可以向 Docker 传递参数来禁用不同类型的隔离，要么是为了提高容器的灵活性，要么是为了性能。

+   你可以禁用单个容器的 Linux OOM 杀手，以指示 Linux 永远不会通过杀死此进程来尝试回收有限的内存。

+   nsenter 可以用来从主机获取容器的网络上下文。

+   tcpflow 允许你监控容器内外所有流量，而无需重新配置或重启任何东西。

+   strace 是识别为什么 Docker 容器在特定主机上不起作用的一个关键工具。

这本书到此结束！我们希望我们已经打开了你的眼界，让你看到了 Docker 的一些用途，并给你提供了一些将其整合到你的公司或个人项目中的想法。如果你想与我们联系或给我们一些反馈，请在 Manning *Docker in Practice* 论坛([`forums.manning.com/forums/docker-in-practice-second-edition`](https://forums.manning.com/forums/docker-in-practice-second-edition))中创建一个帖子，或者针对“docker-in-practice”GitHub 仓库中的一个创建一个问题。
