## 附录 B. Docker 配置

在本书的各个部分，你被建议更改 Docker 配置，以便在启动 Docker 主机时使更改永久化。附录 B 将为你提供实现此目的的最佳实践建议。你使用的操作系统分发版在此背景下将非常重要。

## 配置 Docker

大多数主流分发版的配置文件位置列在 表 B.1 中。

##### 表 B.1\. Docker 配置文件位置

| **分发** | **配置** |
| --- | --- |
| Ubuntu, Debian, Gentoo | /etc/default/docker |
| OpenSuse, CentOS, Red Hat | /etc/sysconfg/docker |

注意，一些分发版将配置保留为单个文件，而其他分发版则使用目录和多个文件。例如，在 Red Hat Enterprise License 上，有一个名为 /etc/sysconfig/docker/docker-storage 的文件，按照惯例，它包含与 Docker 守护进程存储选项相关的配置。

如果你的分发版中没有与表 B.1 中列出的名称匹配的文件，那么检查 /etc/docker 文件夹是值得的，因为那里可能存在相关的文件。

在这些文件中，管理 Docker 守护进程启动命令的参数。例如，当编辑时，以下类似的行允许你为主机上的 Docker 守护进程设置启动参数。

```
DOCKER_OPTS=""
```

例如，如果你想将 Docker 的根目录位置从默认位置（即 /var/lib/docker）更改为其他位置，你可能需要将前面的行更改为以下内容：

```
DOCKER_OPTS="-g /mnt/bigdisk/docker"
```

如果你的分发版使用 systemd 配置文件（而不是 /etc），你还可以在 systemd 文件夹下的 docker 文件中搜索 `ExecStart` 行，并根据需要更改它。例如，该文件可能位于 /usr/lib/systemd/system/service/docker 或 /lib/systemd/system/docker.service。以下是一个示例文件：

```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io
After=network.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/docker
ExecStart=/usr/bin/docker -d --selinux-enabled
Restart=on-failure
LimitNOFILE=1048576
LimitNPROC=1048576

[Install]
WantedBy=multi-user.target
```

`EnvironmentFile` 行将启动脚本指向我们之前讨论的 `DOCKER_OPTS` 条目的文件。如果你直接更改 systemctl 文件，你需要运行 `systemctl daemon-reload` 来确保更改被 systemd 守护进程拾取。

## 重启 Docker

仅更改 Docker 守护进程的配置是不够的——为了应用更改，守护进程必须重启。请注意，这将停止任何正在运行的容器并取消任何正在进行的镜像下载。

***使用 systemctl 重启***

大多数现代 Linux 分发版使用 systemd 来管理机器上服务的启动。如果你在命令行上运行 `systemctl` 并得到一页的输出，那么你的主机正在运行 systemd。如果你得到“命令未找到”的消息，请转到下一节。

如果你想更改配置，你可以按照以下步骤停止并启动 Docker：

```
$ systemctl stop docker
$ systemctl start docker
```

或者，你也可以直接重启：

```
$ systemctl restart docker
```

通过运行以下命令来检查进度：

```
$ journalctl -u docker
$ journalctl -u docker -f
```

这里第一行输出了 docker 守护进程的可用日志。第二行跟随任何新的条目。

***使用服务重启***

如果您的系统正在运行基于 System V 的 init 脚本集，请尝试运行 `service --status-all`。如果它返回服务列表，您可以使用 `service` 命令以新的配置重启 Docker。

```
$ service docker stop
$ service docker start
```
