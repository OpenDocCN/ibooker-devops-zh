# 10 在 Docker Swarm 上运行的云原生应用程序

本章涵盖了

+   在 AWS 上部署自修复的 Swarm 集群并使用 S3 存储桶进行节点发现

+   在 Jenkins 管道中运行基于 SSH 的命令并配置 SSH 代理

+   自动部署 Docker 化应用程序到 Swarm

+   将 Slack 集成到 CI/CD 管道的发布和构建通知管理中

+   在 Jenkins 中进行持续交付到生产以及用户手动批准

上一章介绍了如何使用 Jenkins 为容器化的微服务应用程序设置持续集成管道。本章将介绍如何自动化部署和管理多个应用程序环境。在本章结束时，你将熟悉在 Docker Swarm 集群中运行的容器化微服务的持续部署和交付（图 10.1）。

![](img/CH10_F01_Labouardy.png)

图 10.1 完整的 CI/CD 管道工作流程

在多台机器上运行多个容器的基本解决方案之一是 Swarm ([`docs.docker.com/engine/swarm/`](https://docs.docker.com/engine/swarm/))，它捆绑在 Docker 引擎中。在本章结束时，你应该能够从头开始构建一个 CI/CD 管道，用于在 Docker Swarm 集群内部运行的服务，如图 10.2 所示。

![](img/CH10_F02_Labouardy.png)

图 10.2 目标 CI/CD 管道

## 10.1 运行分布式 Docker Swarm 集群

Docker Swarm 最初作为一个独立产品发布，在服务器集群上运行主容器和代理容器以编排容器的部署。2016 年 Docker 1.12 版本的发布改变了这一点。Docker Swarm 成为 Docker 引擎的官方部分，并直接集成到每个 Docker 安装中。

注意：这只是 Docker 中 Docker Swarm 功能的简要概述。欲了解更多信息，请自由探索 Docker Swarm 的官方文档([`docs.docker.com/engine/swarm/`](https://docs.docker.com/engine/swarm/))。

为了说明从 Jenkins 中定义的 CI/CD 管道将容器部署到 Swarm 集群的过程，我们需要部署一个 Swarm 集群。

Swarm 集群将在一个 VPC 内部部署，包含两个自动扩展组：一个用于 Swarm 管理器，另一个用于 Swarm 工作节点。这两个 ASG 都将在多个可用区中启动的私有子网内部部署，以提高弹性。

一旦创建了 ASGs，设置 Swarm 需要手动初始化管理器，并且将新节点添加到集群中需要额外的信息（一个集群加入令牌），这是在创建 Swarm 时由第一个管理器提供的。

此步骤可以使用 Ansible 或 Chef 等配置管理工具自动化。然而，它需要手动交互。为了解决这个问题，并提供自动 Swarm 初始化，我们将在实例启动时运行一个单次 Docker 容器；该容器使用 S3 存储桶作为集群发现注册表以找到活动管理器和加入令牌。

图 10.3 总结了我们将部署的架构。我们将专注于 AWS，但相同的架构也可以应用于其他云提供商或本地环境。

![图片](img/CH10_F03_Labouardy.png)

图 10.3 AWS 中的 Swarm 架构

注意：可以使用分布式、一致性的键值存储，如 etcd([`etcd.io/`](https://etcd.io/))、HashiCorp 的 Consul([www.consul.io](http://www.consul.io))或 Apache ZooKeeper([`zookeeper.apache.org/`](https://zookeeper.apache.org/))作为服务发现，使节点自动加入 Swarm 集群。

要部署 Swarm 实例，我们需要提供一个预安装了 Docker Engine 的 AMI。到目前为止，你应该熟悉 Packer。我们将创建一个包含以下列表内容的 template.json 文件。(完整的模板可以从 chapter10/swarm/packer/docker-ce/template.json 下载。)

列表 10.1 Docker AMI 的 Packer 模板

```
{
    "variables" : {},
    "builders" : [
        {
            "type" : "amazon-ebs",
            "profile" : "{{user `aws_profile`}}",
            "region" : "{{user `region`}}",
            "instance_type" : "{{user `instance_type`}}",
            "source_ami" : "{{user `source_ami`}}",
            "ssh_username" : "ec2-user",
            "ami_name" : "18.09.9-ce",
            "ami_description" : "Docker engine AMI",
        }
    ],
    "provisioners" : [
        {
            "type" : "shell",
            "script" : "./setup.sh",
            "execute_command" : "sudo -E -S sh '{{ .Path }}'"
        }
    ]
}
```

基础镜像为 Amazon Linux 2，它将通过一个 shell 脚本安装最新的 Docker 社区版软件包。然后它将`ec2-user`用户名添加到`docker`组中，以便能够在不使用`sudo`命令的情况下执行 Docker 命令；请参阅以下列表。

列表 10.2 Docker 社区版安装

```
#!/bin/bash
yum update -y
yum install docker -y
usermod -aG docker ec2-user
systemctl enable docker
```

执行`packer build`命令来烘焙 Docker AMI。一旦配置过程完成，新的烘焙 AMI 应该在 AWS 管理控制台上的“图像”部分可用（图 10.4）。

![图片](img/CH10_F04_Labouardy.png)

图 10.4 Docker 社区版 AMI

接下来，使用 Terraform 部署基础设施，并创建一个名为`sandbox`的专用 VPC，使用 10.1.0.0/16 CIDR 块来隔离沙盒应用程序和工作负载。在 vpc.tf 文件中定义列表 10.3 中的块。

注意：在不同的 VPC 上部署集群不是强制性的，但强烈建议遵循最佳实践，通过隔离工作负载环境以进行审计和安全合规性。

列表 10.3 沙盒 VPC 资源

```
resource "aws_vpc" "sandbox" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true

  tags = {
    Name   = var.vpc_name
    Author = var.author
  }
}
```

Swarm 管理器需要在初始化后有一种方式将工作令牌传递给工作节点。最佳方式是让 Swarm 管理器的用户数据触发生成令牌并将其放入 S3 桶中。在 s3.tf 中使用以下列表中的代码定义一个私有 S3 桶资源。

列表 10.4 Swarm 发现 S3 桶资源

```
resource "aws_s3_bucket" "swarm_discovery_bucket" {
  bucket = var.swarm_discovery_bucket
  acl    = "private"

  tags = {
    Author = var.author
    Environment = var.environment
  }
}
```

注意：AWS 系统管理器参数存储([`mng.bz/r6GX`](https://shortener.manning.com/r6GX))也可以用作共享加密存储，用于存储和检索 Swarm 工作节点的加入令牌。

IAM 实例配置文件对于 EC2 实例能够与 S3 桶交互以存储或检索用于自动加入操作的工作令牌是必要的。在 iam.tf 文件中定义 IAM 角色策略，如下所示。

列表 10.5 Swarm 节点 IAM 策略

```
resource "aws_iam_role_policy" "discovery_bucket_access_policy" {
  name = "discovery-bucket-access-policy-${var.environment}"
  role = aws_iam_role.swarm_role.id

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOF
}
```

然后，我们为 Swarm 管理器创建一个启动配置，该配置使用 Packer 烤制的 Docker AMI，并在用户数据上运行配置的启动脚本。使用以下列表在 swarm_managers.tf 中定义代码。

列表 10.6 Swarm 管理器启动配置

```
resource "aws_launch_configuration" "managers_launch_conf" {
  name                 = "managers_config_${var.environment}"
  image_id             = data.aws_ami.docker.id
  instance_type        = var.manager_instance_type
  key_name             = var.key_name
  security_groups      = [aws_security_group.swarm_sg.id]
  user_data            = data.template_file.swarm_manager_user_data.rendered
  iam_instance_profile = aws_iam_instance_profile.swarm_profile.id

  root_block_device {
    volume_type = "gp2"
    volume_size = 20
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

启动脚本使用集群发现 S3 存储桶的名称和运行实例的角色（管理器或工作节点），如下所示。根据实例角色，`docker` `swarm` `join` 命令将使用正确的令牌（`workers` 令牌或 `managers` 令牌）。

列表 10.7 Swarm 管理器用户数据

```
data "template_file" "swarm_manager_user_data" {
  template = "${file("scripts/join-swarm.tpl")}"
  vars = {
    swarm_discovery_bucket = "${var.swarm_discovery_bucket}"
    swarm_name             = var.environment
    swarm_role             = "manager"
  }
}
```

如下所示，shell 脚本 [joint-swarm.tpl](https://plugins.jenkins.io/credentials-binding/) 使用 EC2 元数据获取实例的私有 IP 地址。脚本随后执行一个容器，该容器使用 S3 存储桶存储 Swarm 的状态，一旦创建或如果存储桶中不存在状态，则创建一个新的 Swarm。

列表 10.8 Swarm 节点启动脚本

```
#!/bin/bash
NODE_IP=$(curl -fsS http://169.254.169.254/latest/meta-data/local-ipv4)
docker run -d --restart on-failure:5 \
    -e SWARM_DISCOVERY_BUCKET=${swarm_discovery_bucket} \
    -e ROLE=${swarm_role} \
    -e NODE_IP=$NODE_IP \
    -e SWARM_NAME=${swarm_name} \
    -v /var/run/docker.sock:/var/run/docker.sock \
    mlabouardy/swarm-discovery
```

注意：您可以使用 CloudWatch 告警定义自动扩展策略，根据 CPU 利用率或 Swarm 节点的自定义指标触发扩展或缩减事件。

从那里，我们将创建一个管理器的自动扩展组（ASG）。默认情况下，我们将为集群创建一个管理器。但是，我建议在生产环境中使用奇数，因为管理器之间需要多数投票来就提议的管理任务达成一致。奇数（而不是偶数）强烈推荐用于打破平局。然而，对于沙盒集群，我们将保持简单，使用一个 Swarm 管理器。在 swarm_mangers.tf 中，定义如下所示的 ASG 资源。

表 10.1 Swarm 集群安全组规则

```
resource "aws_autoscaling_group" "swarm_managers" {
  name                 = "managers_asg_${var.environment}"
  launch_configuration = aws_launch_configuration.managers_launch_conf.name
  vpc_zone_identifier  = [for subnet in aws_subnet.private_subnets: subnet.id]
  depends_on = [aws_s3_bucket.swarm_discovery_bucket]
  min_size       = 1
  max_size       = 3
  lifecycle {
    create_before_destroy = true
  }
}
```

列表 10.10 Swarm 工作节点 ASG

同样，我们将为工作节点创建一个 ASG，我们将使用两个 Swarm 工作节点。注意使用 `depends_on` 关键字创建对 `swarm_managers` 资源的隐式依赖。Terraform 使用此信息来确定创建资源的正确顺序。

| 协议 | 端口 | 来源 | 描述 |

列表 10.9 Swarm 管理器自动扩展组

```
resource "aws_autoscaling_group" "swarm_workers" {
  name                 = "workers_asg_${var.environment}"
  launch_configuration = aws_launch_configuration.workers_launch_conf.name
  vpc_zone_identifier  = [for subnet in aws_subnet.private_subnets: subnet.id]
  min_size             = 2
  max_size             = 5
  depends_on = [aws_autoscaling_group.swarm_managers]
  lifecycle {
    create_before_destroy = true
  }
}
```

最后，允许分配给 Swarm 集群实例的安全组中的表 10.1 中的防火墙规则。

| TCP | 7946 | Swarm | 所有节点之间的控制平面 Gossip 发现通信 |

| TCP | 2377 | Swarm | 集群管理和 raft 同步通信 |
| --- | --- | --- | --- |
| 注意：mlabouardy/swarm-discovery 完整的 Python 脚本和 Dockerfile 在 GitHub 仓库中给出：pipeline-as-code-with-jenkins/tree/master/chapter10/discovery。 |
| UDP | 7946 | Swarm | 来自其他 Swarm 节点的容器网络发现 |
| 协议 | 端口 | 来源 | 描述 |
| 在此示例中，Terraform 将首先创建 Swarm 管理器。这样，我们保证 Swarm 初始化和 S3 存储桶中存在加入令牌的可用性。在 swarm_workers.tf 文件中添加以下列表中的资源。 |
| TCP | 22 | Jenkins 和防火墙 SG | 来自 Jenkins 主和防火墙安全组的 SSH 流量 |

以下列表提供了安全组定义。

列表 10.11 Swarm 节点安全组

```
resource "aws_security_group" "swarm_sg" {
  name        = "swarm_sg_${var.environment}"
  description = "Allow inbound traffic fo.
swarm management and ssh from jenkins & bastion hosts"
  vpc_id      = aws_vpc.sandbox.id

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [var.bastion_sg_id, var.jenkins_sg_id]
  }
  ingress {
    from_port   = "2377"
    to_port     = "2377"
    protocol    = "tcp"
    cidr_blocks = [var.cidr_block]
  }
  ...
  egress {
    from_port   = "0"
    to_port     = "0"
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

注意：我建议使用启用加密和版本控制的 S3 后端来远程存储 Terraform 状态文件。

在`variables.tfvars`中定义所需的 Terraform 变量，如表 10.2 所示。

表 10.2 Swarm Terraform 变量

| 变量 | 类型 | 值 | 描述 |
| --- | --- | --- | --- |
| `region` | 字符串 | None | 部署 Swarm 集群的区域的名称，例如`eu-central-1` |
| `shared_credentials_file` | 字符串 | `~/.aws/credentials` | 共享凭据文件的路径。如果没有设置且指定了配置文件，则使用`~/.aws/credentials`。 |
| `aws_profile` | 字符串 | `profile` | 在共享凭据文件中设置的 AWS 配置文件名称 |
| `author` | 字符串 | None | Swarm 集群所有者的名称。标记您的 AWS 资源以跟踪按所有者或环境划分的月度成本是可选的，但建议这么做。 |
| `key_name` | 字符串 | None | SSH 密钥对 |
| `availability_zones` | 列表 | None | 启动 VPC 子网的可用区 |
| `bastion_sg_id` | 字符串 | None | 防火墙主机安全组的 ID |
| `jenkins_sg_id` | 字符串 | None | Jenkins 主安全组的 ID |
| `vpc_name` | 字符串 | `sandbox` | VPC 的名称 |
| `environment` | 字符串 | `sandbox` | 运行时环境名称 |
| `cidr_block` | 字符串 | `10.1.0.0/16` | VPC 的 CIDR 块 |
| `cluster_name` | 字符串 | `sandbox` | Swarm 集群的名称 |
| `public_subnets_count` | 数字 | 2 | 需要创建的公共子网数量 |
| `private_subnets_count` | 数字 | 2 | 需要创建的私有子网数量 |
| `swarm_discovery_bucket` | 字符串 | `swarm-discovery-cluster` | 存储 Swarm 令牌的 S3 存储桶 |
| `manager_instance_type` | 字符串 | `t2.small` | Swarm 管理员的 EC2 实例类型 |
| `worker_instance_type` | 字符串 | `t2.large` | Swarm 工作节点的 EC2 实例类型 |

然后，使用`terraform apply`命令开始部署过程。一旦部署完成，将创建自动扩展组（ASGs），在每个实例上启动 Swarm 发现容器，第一个运行的经理将执行`swarm init`命令并将令牌存储在 S3 存储桶中（图 10.5），其他实例将使用此令牌加入集群。

注意：您可以拥有任意数量的工作节点组，运行在您选择的任意配置中（CPU 或内存优化的工作节点与通用 Swarm 工作节点一起运行）。

![图片](img/CH10_F05_Labouardy.png)

图 10.5 存储在 S3 存储桶中的 Swarm 状态

如果您决定为 Swarm 集群创建一个专用的 VPC，您需要设置管理和沙盒 VPC 之间的 VPC 对等连接，如图 10.6 所示。有关如何使用 Terraform 设置对等的分步指南，请参阅官方 Terraform 文档，网址为 [`mng.bz/VBw5`](http://mng.bz/VBw5)。

![图 10.11 创建部署仓库的分支](img/CH10_F06_Labouardy.png)

图 10.6 管理和沙盒 VPC 之间的 VPC 对等连接

注意：如果您打算使用 VPC 对等连接，请确保 VPC 不具有匹配或重叠的 IPv4 CIDR 块。在我们的示例中，管理和沙盒 CIDR 块分别为 10.0.0.0/16 和 10.1.0.0/16。

从 VPC 仪表板导航到对等连接，创建一个新的对等连接。配置对等连接，如图 10.7 所示。

![图 10.7 配置管理和沙盒 VPC 的对等连接](img/CH10_F07_Labouardy.png)

图 10.7 配置管理和沙盒 VPC 的对等连接

创建对等连接后，您将在状态栏中看到“待接受”。如果您使用的是不同的账户或不同的区域，请转到相应的 VPC 控制台，在那里您可以在对等连接的状态栏中看到“待接受”。从“操作”下拉菜单中选择“接受请求”，如图 10.8 所示。然后，在“接受 VPC 对等连接请求”提示框中，单击“是，接受”。

![图 10.10 Swarm 集群节点列表](img/CH10_F08_Labouardy.png)

图 10.8 接受 VPC 对等连接请求

要通过此 VPC 对等连接发送和接收流量，您必须在您的 VPC 路由表中添加一个路由到对等 VPC。在关联 VPC 子网的路由表中，创建一个以对等 VPC 的 CIDR 块为目标，以 VPC 对等连接的 ID 为目标的路由。

对所有其他 VPC 路由表重复相同的设置。一旦设置完成，您的路由表将类似于图 10.9。

![图 10.7 配置管理和沙盒 VPC 的对等连接](img/CH10_F09_Labouardy.png)

图 10.9 沙盒 VPC 的路由表更新

要查看 Swarm 状态，请使用第五章第 5.2.4 节中部署的堡垒主机设置 SSH 隧道：

```
ssh -N 3000:SWARM_MANAGER_IP:22 ec2-user@BASTION_IP
ssh ec2-user@localhost -p 3000
```

将 `SWARM_MANAGER_IP` 替换为 Swarm 管理器的私有 IP 地址。一旦连接，如果您输入 `docker info` 命令，`Swarm:` `active` 属性应确认 Swarm 已正确配置：

![图 10.10 Swarm 的连接节点](img/CH10_F09_UN01_Labouardy.png)

从管理机器上运行 `docker node ls` 命令以查看您的 Swarm 的连接节点。如图 10.10 所示，我们现在有一个管理节点和两个工作节点。

```
docker node ls
```

![图 10.10 Swarm 的连接节点](img/CH10_F10_Labouardy.png)

图 10.10 Swarm 集群节点列表

随着 Swarm 运行起来，让我们使用 Jenkins 部署基于 Docker 的应用程序。

## 10.2 定义持续部署流程

为部署创建一个新的 GitHub 仓库。由于部署选项经常更改，我们将部署部分存储在不同的 Git 仓库中。然后，创建三个主要分支：develop、preprod 和 master，如图 10.11 所示。

Docker Swarm 模式现在直接集成到 Docker Compose v3，并正式支持通过 docker-compose.yml 文件部署 *stacks*（服务组）。您用于在本地测试应用程序的相同的 docker-compose.yml 文件现在可以用于将应用程序部署到 Swarm。

![图片](img/CH10_F11_Labouardy.png)

图 10.11 GitHub 部署仓库

要从 Jenkins 进行 Docker Swarm 部署，我们需要一个包含 Docker 镜像引用以及配置设置（如端口、网络名称、标签和约束）的 docker-compose 文件。要运行此文件，我们需要在管理机器上通过 SSH 执行 `docker stack deployment` 命令。

在 develop 分支上，使用您喜欢的文本编辑器或 IDE 创建 docker-compose.yml 文件，内容如下所示。

列表 10.12 应用 Docker Compose

```
version: "3.3"
services:
  movies-loader:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-loader:develop
    environment:
      - AWS_REGION=REGION
      - SQS_URL=https://sqs.REGION.amazonaws.com/ID/movies_to_parse_sandbox

  movies-parser:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-loader:develop
    environment:
      - AWS_REGION=REGION
      - SQS_URL=https://sqs.REGION.amazonaws.com/ID/movies_to_parse_sandbox
      - MONGO_URI=mongodb://root:root@mongodb/watchlist
      - MONGO_DATABASE=watchlist
    depends_on:
      - mongodb

  movies-store:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-store:develop
    environment:
      - MONGO_URI=mongodb://root:root@mongodb/watchlist
    ports:
      - 3000:3000
    depends_on:
      - mongodb

  movies-marketplace:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-marketplace:develop
    ports:
      - 80:80

  mongodb:
    image: bitnami/mongodb:latest
    environment:
      - MONGODB_USERNAME=root
      - MONGODB_PASSWORD=root
      - MONGODB_DATABASE=watchlist
```

注意：将 `ID`、`REGION` 和 `USER` 替换为您的 AWS 账户 ID、AWS 区域和 ECR URI。

每个服务都使用我们在第九章中构建的镜像，并引用 `develop` 标签。此标签专门用于沙盒部署，包含 develop 分支的代码库。此外，我们还定义了一个 MongoDB 服务，该服务将被 movies-store 和 movies-parser 服务使用。

MongoDB 服务的凭据是明文。然而，在任何情况下都不应提交敏感信息，并选择像 HashiCorp Vault 或 AWS SSM Parameter Store 这样的托管解决方案来加密您的凭据和访问令牌。您还可以使用 Docker 的集成功能 Secrets 创建数据库凭据：

```
openssl rand -base64 12 | docker secret create mongodb_password -
```

并更新 docker-compose.yml 以使用密钥而不是明文密码：

```
mongodb:
    image: bitnami/mongodb:latest
    environment:
      - MONGODB_USERNAME=root
      - MONGO_ROOT_PASSWORD_FILE: /run/secrets/mongodb_password
      - MONGODB_DATABASE=watchlist
```

注意：如果 MongoDB 服务因未知原因崩溃或已被删除，其数据将丢失。为了避免数据丢失，您应该挂载一个持久卷。根据所使用的云提供商，Docker 卷支持使用外部持久存储，如 Amazon EBS。

为了解耦 HTML 页面的爬取和解析，我们在 movies-loader 和 movies-parser 服务之间使用分布式队列。除了其高可用性外，这还将允许我们根据要解析的 HTML 页面数量部署额外的 movies-parser 工作节点。使用 Terraform（chapter10/swarm/terraform/sqs.tf）创建一个名为 `movies_to_parse_sandbox` 的沙盒环境 SQS，如图 10.12 所示。此队列将由 movies-loader 用于推送电影，然后它将被 movies-parser 工作节点消费。

![图片](img/CH10_F12_Labouardy.png)

图 10.12 沙盒队列设置

在处理完 Docker Compose 后，我们可以继续创建 Jenkinsfile，如列表 10.13 所示，包含以下步骤：

1.  克隆 GitHub 仓库（chapter10/deployment/sandbox/Jenkinsfile）并检出 develop 分支。

1.  通过 SSH 将 docker-compose.yml 文件发送到管理节点并执行 `docker stack deploy` 命令。

注意：我们使用 master 标签来限制管道仅在 Jenkins 主节点上执行。工作机的机器也可能用于此作业。

列表 10.13 部署 Jenkinsfile

```
def swarmManager = 'manager.sandbox.domain.com'
def region = 'AWS REGION'                           ❶
node('master'){
    stage('Checkout'){
        checkout scm
    }
    stage('Copy'){
        sh "scp -o StrictHostKeyChecking=no
docker-compose.yml ec2-user@${swarmManager}:/home/ec2-user"
    }
    stage('Deploy stack'){
        sh "ssh -oStrictHostKeyChecking=no ec2-user@${swarmManager} '\$(\$(aws ecr get-login --no-include-email --region ${region}))' || true"
        sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager} docker stack deploy
--compose-file docker-compose.yml
--with-registry-auth watchlist"
    }
}
```

❶ 替换为你的 AWS 默认区域。

此 Jenkinsfile 使用 Amazon ECR 作为私有仓库。如果你使用的是需要用户名和密码认证的私有仓库（例如 Nexus、DockerHub、Azure 或 Cloud Container Registry），你可以使用默认安装的 Credentials Binding 插件（[`plugins.jenkins.io/credentials-binding/`](https://plugins.jenkins.io/credentials-binding/)），将仓库凭据绑定到`USERNAME`和`PASSWORD`变量。然后，将这些变量传递给`docker login`命令进行认证：

```
stage('Deploy'){
    withCredentials([[
$class: 'UsernamePasswordMultiBinding',
credentialsId: 'registry',
usernameVariable: 'USERNAME',
passwordVariable: 'PASSWORD']]) {
        sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
docker login --password $PASSWORD --username $USERNAME
${registry}"
        sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
docker stack deploy --compose-file docker-compose.yml
--with-registry-auth watchlist"
    }
}
```

使用以下命令将 Jenkinsfile 和 docker-compose.yml 文件推送到 develop 分支：

```
git add . 
git commit -m "deploy watchlist stack to sandbox"
git push origin develop
```

前往 Jenkins，创建一个名为 watchlist-deployment 的新多分支管道作业。

注意：有关如何在 Jenkins 上创建和配置多分支管道作业的逐步指南，请参阅第七章。

设置 GitHub 仓库 HTTPS 克隆 URL，并允许 Jenkins 发现所有分支，寻找根仓库中的 Jenkinsfile，如图 10.13 所示。

![图片](img/CH10_F13_Labouardy.png)

图 10.13 分支源配置

目前，作业管道应该发现 develop 分支并执行 Jenkinsfile 中定义的阶段，如图 10.14 所示。

![图片](img/CH10_F14_Labouardy.png)

图 10.14 Jenkins 上的部署作业

管道应该在复制阶段失败并变成红色，如图 10.15 所示。Jenkins 主节点无法 SSH 到 Swarm 管理器，因为 Jenkins 主节点有错误的私钥。

![图片](img/CH10_F15_Labouardy.png)

图 10.15 SCP 命令日志

为了让 Jenkins 持续部署到 Swarm，它需要访问 Swarm 管理器。在 Jenkins 上创建一个新的 SSH 用户名凭据，带有私钥，以访问 Swarm 沙盒。在私钥字段中粘贴创建 Swarm EC2 实例时使用的密钥对的内容。然后，将其命名为`swarm-sandbox`，如图 10.16 所示。

![图片](img/CH10_F16_Labouardy.png)

图 10.16 Jenkins 凭据与 Swarm SSH 密钥对

注意：Jenkins 只需要访问 Swarm 管理器。其他节点由 Swarm 管理器管理，因此 Jenkins 不需要直接访问它们。

更新 Jenkinsfile 以使用 SSH 代理插件（Credentials Binding 插件）注入凭据。`sshagent`块应该包装所有基于 SSH 和 SCP 的命令，如下面的列表所示。

列表 10.14 SSH 代理配置

```
sshagent (credentials: ['swarm-sandbox']){
   stage('Copy'){
     sh "scp -o StrictHostKeyChecking=no
docker-compose.yml ec2-user@${swarmManager}:/home/ec2-user"
   }

   stage('Deploy stack'){
     sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
'\$(\$(aws ecr get-login --no-include-email --region ${region}))'
|| true"
     sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
docker stack deploy --compose-file docker-compose.yml
--with-registry-auth watchlist"
   }
}
```

将更改推送到 develop 分支。watchlist-deployment 项目的 develop 分支的嵌套作业应该触发一个新的构建。

注意：为了实现持续部署，请在 GitHub 仓库上创建一个 GitHub webhook，以便在推送事件时通知 Jenkins。

这次，管道应该成功并变成绿色（图 10.17）。

![图片](img/CH10_F17_Labouardy.png)

图 10.17 持续部署管道

在构建日志方面，Jenkins 将在 Swarm 管理器上通过 SSH 运行`docker` `stack` `deploy`，并根据`develop`标签图像部署图 10.18 中的服务。

![图片](img/CH10_F18_Labouardy.png)

图 10.18 `docker stack deploy`的输出

注意：如果您计划将 Amazon ECR 用作远程存储库，您需要将 ECR IAM 策略分配给分配给 Swarm 实例的 IAM 实例配置文件。

在 Swarm 上，输入以下命令，我们应该能够查看堆栈的状态以及其中运行的服务：

```
docker service ls
```

四个微服务应与 MongoDB 服务一起部署，如图 10.19 所示。

![图片](img/CH10_F19_Labouardy.png)

图 10.19 Swarm 沙盒上成功部署的堆栈

接下来，我们将部署一个名为 Visualizer 的开源工具，以可视化跨一组机器的 Docker 服务。在 Swarm 管理器机器上执行以下命令：

```
docker service create --name=visualizer 
--publish=8080:8080/tc.
--constraint=node.role==manager \
     --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
     dockersamples/visualizer
```

一旦服务部署完成，我们将创建一个公共负载均衡器，将传入的 HTTP 和 HTTPS（可选）流量转发到 8080 端口，这是 Visualizer UI 暴露的端口。在以下列表中声明 ELB 资源或从 chapter8/services/loadbalancers.tf 下载资源文件。

列表 10.15 Visualizer 负载均衡器

```
resource "aws_elb" "visualizer_elb" {
  subnets                   = var.public_subnets
  cross_zone_load_balancing = true
  security_groups           = [aws_security_group.elb_visualizer_sg.id]
  listener {
    instance_port      = 8080
    instance_protocol  = "http"
    lb_port            = 443
    lb_protocol        = "https"
    ssl_certificate_id = var.ssl_arn
  }
  listener {
    instance_port      = 8080
    instance_protocol  = "http"
    lb_port            = 80
    lb_protocol        = "http"

  }
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
    target              = "TCP:8080" resource "aws_autoscaling_attachment" "cluster_attach_visualizer_elb" {
  autoscaling_group_name = var.swarm_managers_asg_id
  elb                    = aws_elb.visualizer_elb.id
}

    interval            = 5
  }
}
```

然后，我们将负载均衡器附加到 Swarm 管理器的 ASG。负载均衡器也可以分配给 Swarm 工作节点。实际上，Swarm 集群中的所有节点都通过 gossip 网络知道集群中每个容器的位置。如果传入的请求击中了当前未运行该请求所针对服务的服务的节点，请求将被路由到运行该服务容器的节点。

这样做是为了避免节点需要为特定服务专门构建。任何节点都可以运行任何服务，并且每个节点都可以进行均衡负载，从而减少复杂性和应用程序所需的资源数量。这个特性被称为 *网状路由*：

```
resource "aws_autoscaling_attachment" "cluster_attach_visualizer_elb" {
  autoscaling_group_name = var.swarm_managers_asg_id
  elb                    = aws_elb.visualizer_elb.id
}
```

以下列表（chapter8/services/dns.tf）不是必需的，但可以用来创建一个友好的 DNS 记录，指向 Visualizer 负载均衡器的 FQDN。

列表 10.16 Visualizer DNS 配置

```
resource "aws_route53_record" "visualizer" {
  zone_id = var.hosted_zone_id
  name    = "visualizer.${var.environment}.${var.domain_name}"
  type    = "A"
  alias {
    name                   = aws_elb.visualizer_elb.dns_name
    zone_id                = aws_elb.visualizer_elb.zone_id
    evaluate_target_health = true
  }
}
```

注意：更新 Swarm 集群的安全组，允许来自负载均衡器安全组的 8080 端口进入流量。为 8080 端口添加入站规则，并使用`terraform` `apply`使更改生效。

一旦发布更改，将浏览器指向终端会话中“输出”部分显示的负载均衡器 URL。这个实用的工具，如图 10.20 所示，可以帮助您查看哪些容器正在运行，以及它们在哪些节点上。

注意：此工具仅在 Docker Engine 1.12.0 及以后的 Docker Swarm 模式下工作。它不与单独的 Docker Swarm 项目一起工作。

![图片](img/CH10_F20_Labouardy.png)

图 10.20 可视化仪表板

注意：容器也部署在管理器上。如果您想限制部署到工作节点，请使用带有标签的 Docker 约束。

我们已成功将应用程序堆栈部署到 Swarm。然而，目前部署是手动触发的。最终，我们希望部署作业在每个 CI 管道成功执行结束时运行。

要这样做，请更新 Jenkinsfile（chapter10/pipelines/movies-loader/Jenkinsfile），使用 `build job` 关键字触发外部作业。例如，在 movies-loader Jenkinsfile 中，将以下 `Deploy` 阶段代码块添加到管道末尾：

```
stage('Deploy'){
        if(env.BRANCH_NAME == 'develop'){
            build job: "watchlist-deployment/${env.BRANCH_NAME}"
        }
}
```

将更改提交并推送到功能分支。然后创建一个拉取请求（PR）以合并到开发分支。应在功能分支上触发新的构建，一旦完成，Jenkins 将在 PR 上发布构建状态，如图 10.21 所示。

![](img/CH10_F21_Labouardy.png)

图 10.21 拉取请求构建状态

一旦拉取请求得到验证，我们就将其合并到开发分支，并在此分支上触发新的构建，如图 10.22 所示。

![](img/CH10_F22_Labouardy.png)

图 10.22 movies-loader 项目的 Jenkins CI/CD 管道

在 CI 管道的末尾，将执行部署阶段，并在开发分支上触发 watchlist-deployment，如图 10.23 所示。

![](img/CH10_F23_Labouardy.png)

图 10.23 外部作业触发

这将触发部署作业，该作业将部署堆栈并强制拉取带有 `develop` 标签的新 Docker 镜像。对其他 GitHub 仓库重复相同的流程。最终，如果 CI 成功执行，每个仓库将触发沙盒的部署，如图 10.24 所示。

![](img/CH10_F24_Labouardy.png)

图 10.24 市场 CI/CD 管道执行

注意：在第十一章和第十二章中，我们将介绍如何在 CI/CD 管道中从 Jenkins 运行部署应用的自动化健康检查和集成测试。

到目前为止，我们的应用程序已部署到 Swarm 沙盒环境。要访问应用程序，我们需要创建两个公共负载均衡器：一个用于 API（movies-store）和另一个用于前端（movies-marketplace）。使用 GitHub 仓库中可用的 Terraform 模板文件（位于 /chapter8/services 文件夹下）创建 AWS 资源，然后发出 `terraform apply` 命令以配置资源。在部署过程结束时，市场和企业 API 访问 URL 将在 `输出` 部分显示，如图 10.25 所示。

![](img/CH10_F25_Labouardy.png)

图 10.25 Terraform 应用输出

注意：确保允许来自连接到 Swarm EC2 实例的安全组对端口 80（前端）、8080（可视化器）和 3000（API）的入站流量。

为了使市场能够与 RESTful API 交互并显示爬取的电影列表，我们需要在市场 Docker 镜像构建时注入 API URL。市场源代码根据目标环境包含多个文件（图 10.26）。

![图片](img/CH10_F26_Labouardy.png)

图 10.26 Angular 环境文件

每个文件都包含正确的 API URL。对于沙盒环境，将使用环境.sandbox.ts 文件，如下所示。

列表 10.17 市场沙盒环境变量

```
export const environment = {
  production: false,
  apiURL: 'https://api.sandbox.slowcoder.com',
};
```

市场 Docker 镜像将使用`ng` `build` `-c` `sandbox`标志构建，这将用环境.sandbox.ts 的值替换环境.ts 文件；见图 10.27。

![图片](img/CH10_F27_Labouardy.png)

图 10.27 Docker 镜像构建执行

一旦将新镜像部署到 Swarm，请将您的浏览器指向市场 URL。它应该显示历史上 IMDb 排名前 100 的电影，如图 10.28 所示。

![图片](img/CH10_F28_Labouardy.png)

图 10.28 观看列表市场仪表板

这就是实现持续部署的方法。然而，我们希望提醒开发和产品团队项目的部署和 CI/CD 状态。

## 10.3 将 Jenkins 与 Slack 通知集成

在管道的某些阶段，您可能决定想要向团队发送 Slack 通知，告知他们构建状态。为了通过 Jenkins 发送 Slack 消息，我们需要为我们的作业提供一个授权自身与 Slack 的方式。

幸运的是，Slack 有一个预构建的 Jenkins 集成，使得事情变得相当简单。从[`mng.bz/xXOB`](http://mng.bz/xXOB)安装插件。将`WORKSPACE`替换为如图 10.29 所示的 Slack 工作区名称。

![图片](img/CH10_F29_Labouardy.png)

图 10.29 Jenkins CI Slack 集成

点击“添加到 Slack”按钮。然后选择 Jenkins 发送通知的频道，如图 10.30 所示。

![图片](img/CH10_F30_Labouardy.png)

图 10.30 Slack 频道配置

之后，我们需要在 Jenkins Slack 通知插件（[`plugins.jenkins.io/slack/`](https://plugins.jenkins.io/slack/））上设置配置，该插件已经安装在预制的 Jenkins 主机镜像上。输入团队工作区名称、在您的 Slack 上创建的集成令牌和频道名称，如图 10.31 所示，然后点击应用和保存按钮。

![图片](img/CH10_F31_Labouardy.png)

图 10.31 Jenkins Slack 通知插件

现在我们已经在 Jenkins 中正确配置了 Slack，我们可以配置我们的 CI/CD 管道，使用以下方法发送通知以广播构建状态：

```
slackSend (color: colorCode, message: summary)
```

以电影加载器服务的 CI/CD 管道为例，让我们在管道末尾添加此指令；见以下列表。

列表 10.18 Jenkins Slack 插件 DSL

```
node('workers'){
    stage('Checkout'){}

    stage('Unit Tests'){}

    stage('Build'){}

    stage('Push'){}

    stage('Deploy'){}

    slackSend (color: '#2e7d32',
message: "${env.JOB_NAME} has been successfully deployed")
}
```

注意：为了简化，我跳过了运行单元测试、构建镜像和将镜像推送到注册表的步骤。建议您将这些步骤放入我们即将探索的工作流程中。

将更改推送到功能分支，然后合并到 develop。在管道结束时，将发送一个新的 Slack 通知，如图 10.32 所示。

![图片](img/CH10_F32_Labouardy.png)

图 10.32 Jenkins Slack 通知

虽然这可行，但我们还希望在管道失败时收到通知。这就是 `try-catch` 块发挥作用来处理管道阶段抛出的错误的地方；请参见以下列表。

列表 10.19 Jenkins 中的 Slack 通知

```
node('workers'){
    try {
        stage('Checkout'){
            checkout scm
            notifySlack('STARTED')
        }

        stage('Unit Tests'){}
        stage('Build'){}
        stage('Push'){}
        stage('Deploy'){}
    } catch(e){
        currentBuild.result = 'FAILED'
        throw e
    } finally {
        notifySlack(currentBuild.result)
    }
}
```

这次，使用了 `notifySlack()` 方法，该方法根据管道构建状态发送不同颜色的通知，如以下列表所示。

列表 10.20 自定义 Slack 通知消息颜色

```
def notifySlack(String buildStatus){
    buildStatus =  buildStatus ?: 'SUCCESSFUL'
    def colorCode = '#FF0000'

    if (buildStatus == 'STARTED') {                         ❶
        colorCode = '#546e7a'                               ❶
    } else if (buildStatus == 'SUCCESSFUL') {               ❶
        colorCode = '#2e7d32'                               ❶
    } else {                                                ❶
        colorCode = '#c62828c'                              ❶
    }                                                       ❶
    slackSend (color: colorCode.
message: "${env.JOB_NAME} build status: ${buildStatus}")    ❷
}
```

❶ 在消息的左侧边框上着色

❷ 使用 env.JOB_NAME 发送带有作业名称的 Slack 消息，并使用 buildStatus 变量发送构建状态

根据您的构建结果，代码将发送如图 10.33 所示的 Slack 通知。

![图片](img/CH10_F33_Labouardy.png)

图 10.33 构建状态通知

让我们通过抛出错误来模拟构建失败，将以下指令添加到 `Build` 阶段：

```
error "Build failed"
```

将更改推送到 GitHub。在 `Build` 阶段，管道将失败（见图 10.34）。

![图片](img/CH10_F34_Labouardy.png)

图 10.34 在 Jenkins 管道中抛出错误

在 Slack 频道中，这次我们将收到构建状态设置为失败的通知，如图 10.35 所示。

![图片](img/CH10_F35_Labouardy.png)

图 10.35 构建失败 Slack 通知

在以下列表中，我们将进一步扩展。我们将在通知中添加更多信息，例如推送事件的作者、Git 提交 ID 和消息。

列表 10.21 自定义 Slack 通知消息属性

```
def notifySlack(String buildStatus){
    buildStatus =  buildStatus ?: 'SUCCESSFUL'
    def colorCode = '#FF0000'
    def subject = "Name: '${env.JOB_NAME}'\n
Status: ${buildStatus}\nBuild ID: ${env.BUILD_NUMBER}"       ❶
    def summary = "${subject}\nMessage: ${commitMessage()}
\nAuthor: ${commitAuthor()}\nURL: ${env.BUILD_URL}"          ❷

    if (buildStatus == 'STARTED') {
        colorCode = '#546e7a'
    } else if (buildStatus == 'SUCCESSFUL') {
        colorCode = '#2e7d32'
    } else {
        colorCode = '#c62828c'
    }
    slackSend (color: colorCode, message: summary)
}
```

❶ 显示作业的名称、状态和构建号

❷ 保存主题值和 Git 信息（作者、提交消息）以及构建 URL

`notifySlack()` 方法将调用 `commitAuthor()` 和 `commitMessage()` 来获取适当的信息。`commitAuthor()` 方法将通过执行 `git show` 命令返回提交作者的名称，如以下列表所示。

列表 10.22 获取作者的 Git 辅助函数

```
def commitAuthor(){
    sh 'git show -s --pretty=%an > .git/commitAuthor'           ❶
    def commitAuthor = readFile('.git/commitAuthor').trim()     ❷
    sh 'rm .git/commitAuthor'
    commitAuthor
}
```

❶ 使用 git show 命令显示提交消息的作者，并将输出保存到 commitAuthor 文件中

❷ 读取 commitAuthor 文件并删除额外空格

`commitMessage()` 方法将使用带有 `HEAD` 标志的 `git log` 命令来获取提交消息描述；请参见以下列表。

列表 10.23 获取提交消息的 Git 辅助函数

```
def commitMessage() {
    sh 'git log --format=%B -n 1 HEAD > .git/commitMessage'     ❶
    def commitMessage = readFile('.git/commitMessage').trim()   ❷
    sh 'rm .git/commitMessage'
    commitMessage
}
```

❶ 显示最后提交的消息描述，并将输出保存到 commitMessage 文件中

❷ 读取 commitMessage 内容并删除额外空格

如果我们推送更改，在 CI/CD 管道的末尾，Slack 通知应包含 Jenkins 作业名称、构建 ID 和其状态、作者姓名和提交描述，如图 10.36 所示。

![图片](img/CH10_F36_Labouardy.png)

图 10.36 带有 Git 提交详细信息的 Slack 通知

对 movies-store、movies-marketplace 和 movies-parser 的 Jenkinsfiles 应用相同的更改。

注意：第十一章介绍了如何使用 Jenkins Slack 通知插件发送带有更改日志附件的通知。

## 10.4 使用 Jenkins 处理代码升级

维护多个 Swarm 集群环境在将代码升级到生产时是有意义的，可以避免在生产中破坏东西。同时，拥有类似生产的环境可以帮助您在生产中运行应用程序的镜像，并在预发布环境中重现问题，而不会影响您的客户。但这是有代价的。

注意：您可以通过在正常工作时间之外关闭实例来降低沙箱和预发布环境的成本。

话虽如此，在专用预发布 VPC 中创建一个新的 Swarm 集群用于预发布环境，该 VPC 使用 10.2.0.0/16 CIDR 块，或者将其部署在 Jenkins 部署的同一管理 VPC 中，如图 10.37 所示。

![图片](img/CH10_F37_Labouardy.png)

图 10.37 在同一 VPC 内部署沙箱和预发布 Swarm 集群以及 Jenkins

通过运行以下命令在 watchlist-deployment GitHub 仓库上创建一个预生产分支：

```
git checkout -b preprod
```

创建一个使用 `preprod` 标签的 docker-compose.yml 文件，并将 SQS URL 更新为使用预发布队列，如下所示。

列表 10.24 预发布部署的 Docker Compose

```
version: "3.3"
services:
  movies-loader:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-loader:preprod
    environment:
      - AWS_REGION=eu-west-3
      - SQS_URL=https://sqs.REGION.amazonaws.com/ID/movies_to_parse_staging
  movies-parser:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-parser:preprod
```

使用用于部署 Swarm 预发布集群的 SSH 密钥对创建一个 Jenkins 凭据，类型为 SSH 用户名和私钥。给它命名为 `swarm-staging`，如图 10.38 所示。

![图片](img/CH10_F38_Labouardy.png)

图 10.38 Swarm 预发布集群 SSH 凭据

创建一个类似于 develop 分支中的 Jenkinsfile，如下所示。更新 `swarmManager` 变量以引用预发布的 IP 或 DNS 记录。同时更新 SSH 代理凭据以使用 Swarm 预发布凭据。

列表 10.25 用于预发布部署的 Jenkinsfile

```
def swarmManager = 'manager.staging.domain.com'                  ❶
def region = 'AWS REGION'                                        ❷

node('master'){
    stage('Checkout'){
        checkout scm
    }

     sshagent (credentials: ['swarm-staging']){
         stage('Copy'){
            sh "scp -o StrictHostKeyChecking=no
docker-compose.yml ec2-user@${swarmManager}:/home/ec2-user"      ❸
        }

        stage('Deploy stack'){
            sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
'\$(\$(aws ecr get-login --no-include-email --region ${region})).
|| true"                                                         ❹
            sh "ssh -oStrictHostKeyChecking=no
ec2-user@${swarmManager}
docker stack deploy --compose-fil.
docker-compose.yml --with-registry-auth watchlist"               ❹
        }
     }
}
```

❶ Swarm 管理器 DNS 别名记录或私有 IP 地址

❷ 创建 ECR 存储库的 AWS 区域

❸ 通过 SSH 将 docker-compose.yml 复制到 Swarm 管理实例

❹ 使用 ECR 进行身份验证并通过 SSH 重新部署应用程序堆栈

将更改推送到预生产分支。在 Jenkins 上，当推送事件发生时，应在 watchlist-deployment 项目上触发一个新的预生产嵌套作业，如图 10.39 所示。

![图片](img/CH10_F39_Labouardy.png)

图 10.39 预发布上的堆栈部署

在管道的末尾，应用程序堆栈将被部署到 Swarm 预发布。同样，要访问应用程序，使用 Terraform 部署一个用于市场的公共负载均衡器和商店 API 的公共负载均衡器。

最后，为了在 preprod 上触发自动部署，我们需要为每个项目更新 Jenkinsfile，以触发 preprod 上的 watchlist-deployment——例如，对于 movies-loader Jenkinsfile。我们构建并推送带有 `preprod` 标签的 Docker 镜像，如下一列表所示。

列表 10.26 基于 Git 分支标记 Docker 镜像

```
stage('Push'){
    sh "\$(aws ecr get-login --no-include-email --region ${region}) || true" ❶
    docker.withRegistry("https://${registry}") {
        docker.image(imageName).push(commitID())                             ❷
        if (env.BRANCH_NAME == 'develop') {                                  ❸
            docker.image(imageName).push('develop')                          ❸
        }                                                                    ❸
        if (env.BRANCH_NAME == 'preprod') {                                  ❸
            docker.image(imageName).push('preprod')                          ❸
        }                                                                    ❸
    }
}
```

❶ 使用 AWS CLI 通过 ECR 进行认证

❷ 使用当前的 Git 提交 ID 标记镜像并将其存储在 ECR 中

❸ 基于当前 Git 分支名称，Docker 镜像被标记为唯一的标签。

在以下列表中，我们更新 `Deploy` 阶段的 `if` 子句条件，以在分支名称是 preprod 时触发外部作业的部署。

列表 10.27 触发外部部署作业

```
stage('Deploy'){
    if(env.BRANCH_NAME == 'develop' || env.BRANCH_NAME == 'preprod'){
        build job: "watchlist-deployment/${env.BRANCH_NAME}"
    }
}
```

将更改推送到 develop 分支。然后，在 Jenkins 发布关于 develop 变更的构建状态（如图 10.40）后，创建一个拉取请求以合并 develop 到 preprod 分支。

![图片](img/CH10_F40_Labouardy.png)

图 10.40 拉取请求构建状态

当发生合并时，应该在 preprod 分支上触发一个新的构建，如图 10.41 中的 Blue Ocean 视图所示。

![图片](img/CH10_F41_Labouardy.png)

图 10.41 preprod 分支上的构建触发

一旦执行了 `Push` 阶段，应该将带有 `preprod` 标签的新镜像推送到 Docker 仓库（如图 10.42）。

![图片](img/CH10_F42_Labouardy.png)

图 10.42 存储在 ECR 中的带有 `preprod` 标签的 Docker 镜像

然后，preprod 分支上的部署作业将被执行，以在 Docker Swarm 阶段环境中部署更改（如图 10.43）。

![图片](img/CH10_F43_Labouardy.png)

图 10.43 自动触发的阶段部署

对其他微服务进行相同的更改，除了 movies-marketplace。对于 movies-marketplace，我们需要更新构建阶段，如以下列表所示，以注入适当的环境和将前端指向正确的 API URL。

列表 10.28 在构建过程中注入 API URL

```
stage('Build'){
    switch(env.BRANCH_NAME){
        case 'develop':
            docker.build(imageName, '--build-arg ENVIRONMENT=sandbox .')  ❶
            break
        case 'preprod':
            docker.build(imageName, '--build-arg ENVIRONMENT=staging .')
            break
        default:
            docker.build(imageName)                                       ❷
    }
}
```

❶ 如果分支名称是 develop，我们将环境设置为 sandbox，因此会加载 sandbox 设置。

❷ 如果分支名称不匹配 develop 或 preprod，则默认加载 sandbox 设置。

将更改推送到 GitHub。这次，Docker 构建过程将以 `ENVIRONMENT` 参数设置为 `staging`（当当前分支是 preprod）的方式执行，如图 10.44 所示。这将用 environment .staging.ts 的值替换 environment.ts 文件。

![图片](img/CH10_F44_Labouardy.png)

图 10.44 以环境作为参数的 Docker 构建

## 10.5 实现 Jenkins 交付管道

最后，要将我们的应用程序堆栈部署到生产环境，您需要为生产环境启动一个新的 Swarm 集群。再次，我选择将生产工作负载隔离在专用的生产 VPC 中，该 VPC 使用 10.3.0.0/16 CIDR 块，并在管理 VPC（Jenkins 所在位置）和生产 VPC（Swarm 生产部署位置）之间设置 VPC 对等连接。图 10.45 总结了已部署的架构。

![](img/CH10_F45_Labouardy.png)

图 10.45 多个 Swarm 集群 VPC 的 VPC 对等连接。部署 Jenkins 集群的 VPC 可以访问沙盒、测试和生产 VPC。

注意：VPC 对等连接不支持传递对等连接。生产、测试和沙盒环境是完全隔离的，例如，数据包不能直接从沙盒路由到生产环境，通过管理 VPC 进行路由。

在 watchlist-deployment 仓库的主分支上创建一个 docker-compose .yml 文件。这次，我们使用 `latest` 标签为生产环境中的服务，如下所示。

列表 10.29 生产部署的 Docker Compose。

```
version: "3.3"
services:
  movies-loader:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-loader:latest
    environment:
      - AWS_REGION=eu-west-3
      - SQS_URL=https://sqs.REGION.amazonaws.com/ID/movies_to_parse_production
  movies-parser:
    image: ID.dkr.ecr.REGION.amazonaws.com/USER/movies-parser:latest
```

创建一个 Jenkins 凭据，使用用于部署生产环境 Swarm 集群的 SSH 密钥，并将其命名为 `swarm-production`，如图 10.46 所示。

![](img/CH10_F46_Labouardy.png)

图 10.46 Swarm 生产集群 SSH 凭据。

然后，创建一个 Jenkinsfile，如下所示，以远程上传 docker-compose.yml 文件到管理机器。执行 `docker stack deploy` 命令以部署应用程序。

列表 10.30 生产部署的 Jenkinsfile。

```
def swarmManager = 'manager.production.domain.com'
def region = 'AWS REGION'
node('master'){
    stage('Checkout'){...}                          ❶

     sshagent (credentials: ['swarm-production']){
         stage('Copy'){...}                         ❷

        stage('Deploy stack'){...}                  ❸
     }
}
```

❶ 克隆 GitHub 仓库——有关说明，请参阅 10.25 列表。

❷ 通过 SSH 将 docker-compose.yml 复制到 Swarm 管理器——有关说明，请参阅 10.25 列表。

❸ 通过 SSH 重新部署 Docker Compose 堆栈——有关说明，请参阅 10.25 列表。

将更改推送到主分支。GitHub 仓库应类似于图 10.47。

![](img/CH10_F47_Labouardy.png)

图 10.47 存储在 GitHub 仓库中的部署文件。

Jenkins 管道将在主分支上触发。一旦管道完成，应用程序堆栈将被部署到生产环境，如图 10.48 所示。

![](img/CH10_F48_Labouardy.png)

图 10.48 在主分支中触发的部署。

要在 CI 管道末尾触发生产部署，请更新 GitHub 仓库以触发部署作业，如果当前分支是 master。例如，更新 movies-loader 的 Jenkinsfile 以构建生产镜像，并使用 `latest` 标签将结果推送到 Docker 仓库，如下所示。

列表 10.31 标记生产镜像。

```
stage('Push'){
    sh "\$(aws ecr get-login --no-include-email --region ${region}) || true"
    docker.withRegistry("https://${registry}") {
        docker.image(imageName).push(commitID())
        if (env.BRANCH_NAME == 'develop') {
            docker.image(imageName).push('develop')
        }
        if (env.BRANCH_NAME == 'preprod') {
            docker.image(imageName).push('preprod')
        }
        if (env.BRANCH_NAME == 'master') {
            docker.image(imageName).push('latest')
        }
    }
}
```

对于部署部分，我们可以简单地更新 `if` 子句以支持在主分支上部署：

```
stage('Deploy'){
            if(env.BRANCH_NAME == 'develop.
|| env.BRANCH_NAME == 'preprod.
|| env.BRANCH_NAME == 'master'){
                build job: "watchlist-deployment/${env.BRANCH_NAME}"
            }
}
```

然而，我们希望在部署到生产之前要求手动验证，以模拟产品/业务验证（或 QA 团队在生产批准前运行测试）在部署发布到生产之前。

要这样做，您可以使用输入步骤插件暂停管道执行，并允许用户交互和控制部署过程到生产环境，如下面的列表所示。

列表 10.32 在生产部署前要求用户批准

```
stage('Deploy'){
   if(env.BRANCH_NAME == 'develop' || env.BRANCH_NAME == 'preprod'){
      build job: "watchlist-deployment/${env.BRANCH_NAME}"
   }
   if(env.BRANCH_NAME == 'master'){
      timeout(time: 2, unit: "HOURS") {
         input message: "Approve Deploy?", ok: "Yes"
      }
      build job: "watchlist-deployment/master"
   }
}
```

在这里，我们将超时设置为 2 小时，以给开发者足够的时间验证发布。当 2 小时超时到达时，管道将被终止。

注意：为了避免 Jenkins 工作节点在 2 小时内无所事事，您可以将`Deploy`阶段移出节点块。您还可以在等待用户输入时发送 Slack 提醒。

将更改推送到功能分支，并在功能分支成功构建并被 Jenkins 批准后（图 10.49），提出合并更改到 develop 分支的拉取请求。

![图片](img/CH10_F49_Labouardy.png)

图 10.49 将功能分支合并到 develop 分支

将更改合并到 develop 分支并删除功能分支。应在 develop 分支上触发新的构建，这将部署镜像到 Swarm 沙箱集群；见图 10.50。

![图片](img/CH10_F50_Labouardy.png)

图 10.50 触发沙箱部署

接下来，提出合并 develop 到 preprod 分支的拉取请求（图 10.51）。

一旦 PR 合并，预生产分支将触发新的构建，在 CI/CD 管道的末尾。更改将被部署到 Swarm 预发布集群，如图 10.52 所示。

![图片](img/CH10_F51_Labouardy.png)

图 10.51 将 develop 分支合并到 preprod

![图片](img/CH10_F52_Labouardy.png)

图 10.52 触发到预发布集群的部署

最后，创建一个拉取请求将预生产合并到主分支（图 10.53）。

![图片](img/CH10_F53_Labouardy.png)

图 10.53 将 preprod 分支合并到 master

当合并发生时，Jenkins 将在 movies-loader 服务的 master 分支上触发构建，如图 10.54 所示。然而，这次，一旦达到部署阶段，将弹出部署确认对话框。

![图片](img/CH10_F54_Labouardy.png)

图 10.54 主分支上的 CI/CD 管道执行

正如图 10.55 所示，交互式输入将询问我们是否批准部署。

![图片](img/CH10_F55_Labouardy.png)

图 10.55 部署用户输入对话框

如果我们点击是，管道将被恢复，并在 master 上触发部署作业，如图 10.56 所示。

![图片](img/CH10_F56_Labouardy.png)

图 10.56 生产部署批准

在部署过程结束时，新的堆栈将被部署到 Swarm 生产环境中，并且会向配置的 Slack 频道发送通知（图 10.57）。

![图片](img/CH10_F57_Labouardy.png)

图 10.57 生产部署成功通知

在完成生产部署后，您已经了解了如何将容器化的微服务应用程序部署到多个环境，以及如何在 CI/CD 管道中处理代码升级。然而，因为我们只管理三个环境（沙盒、预发布和生产），我们将通过定义正则表达式将部署作业的发现行为限制为三个主要分支，如图 10.58 所示。

![图片](img/CH10_F58_Labouardy.png)

图 10.58 基于 regular expression 的 Jenkins 发现行为

因此，只有当三个主要分支之一发生变化时，Jenkins 才会发现并触发；见图 10.59。

![图片](img/CH10_F59_Labouardy.png)

图 10.59 部署多分支作业

因此，现在如果我们对我们的应用程序进行任何更改，CI/CD 管道将被触发，并且将执行 `docker` `stack` `deploy`，这将更新任何从上一个版本更改的服务。

注意：如果部署目标是单个主机，则不需要 Swarm。本章中解释的相同的 docker-compose.yml 和程序足以在单主机部署环境中持续部署您的应用程序。

## 摘要

+   可以使用 S3 存储桶或分布式一致性的键值存储，如 etcd、Consul 或 ZooKeeper，作为服务发现，使节点自动加入 Swarm 集群。

+   通过在 Swarm 管理器上通过 SSH 执行 `docker` `stack` `deploy`，可以在 Swarm 集群上实现容器的持续部署。

+   在 CI/CD 管道中添加 Slack 通知可以加快产品交付。团队成员越早意识到构建、集成或部署失败，他们就能越快采取行动。

+   在部署生产版本之前模拟业务/产品验证，Jenkins 输入步骤插件可以在部署前提示用户进行手动验证。
