# 附录 B. 微服务引导速查表

本附录总结了您在本书中学到的最有用的命令。

## B.1 Node.js 命令

请参阅第二章了解如何安装和使用 Node.js 创建微服务。

表 B.1 Node.js 命令（续）

| 命令 | 描述 |
| --- | --- |
| `node --version` | 检查 Node.js 是否已安装并打印版本号。 |
| `npm init -y` | 创建默认的 Node.js 项目。这会为我们的 package.json 创建一个占位符，该文件跟踪 Node.js 项目的元数据和依赖项。 |
| `npm install --save`➥ `<package-name>` | 安装 npm 包。在 npm 上还有许多其他包可用。您可以通过插入特定的包名称来安装任何包。 |
| `npm install` | 安装 Node.js 项目的所有依赖项。这包括所有之前记录在 package.json 中的包。 |
| `node <script-file>` | 运行 Node.js 脚本文件。我们只需调用 `node` 命令，并将脚本文件名作为参数传递。如果您想将其命名为 main.js 或 server.js，也可以，但最好遵守约定，只将其命名为 index.js。 |
| `npm start` | 无论是主脚本文件叫什么名字，还是它期望的命令行参数是什么，`npm start` 都是启动 Node.js 应用的传统 npm 脚本。通常这只是在 package.json 文件中转换为 node index.js，但它完全取决于项目的作者以及他们如何设置。好处是，无论特定项目结构如何，您只需记住 `npm start`。 |
| `npm run start:dev` | 我启动开发中 Node.js 项目的个人约定。我将此添加到 package.json 中的脚本中，通常它运行类似 nodemon 的东西，以便在您与代码一起工作时启用实时重载。 |

## B.2 Docker 命令

请参阅第三章了解如何使用 Docker 打包、发布和运行微服务。

表 B.2 Docker 命令（续）

| 命令 | 描述 |
| --- | --- |
| `docker --version` | 检查 Docker 是否已安装并打印版本号。 |
| `docker container list` | 列出正在运行的容器。 |
| `docker ps` | 列出所有容器（正在运行和已停止的）。 |
| `docker image list` | 列出本地镜像。 |
| `docker build -t <tag> --file`➥ `<docker-file> .` | 根据当前目录中 `docker-file` 中的说明从资产构建一个镜像。`-t` 参数使用您指定的名称标记镜像。 |
| `docker run -d -p`➥ `<host-port>:<container-port>`➥ `<tag>` | 从镜像实例化一个容器。如果镜像在本地不可用，您可以从远程仓库拉取它（假设标签指定了仓库的 URL）。`-d` 参数以分离模式运行容器；它不会绑定到终端，您将看不到输出。省略此参数可以直接看到输出，但请注意，这也会锁定您的终端。`-p` 参数允许您将主机上的端口绑定到容器中的端口。 |
| `docker logs <container-id>` | 从特定容器检索输出。您需要发出此命令才能在分离模式下运行容器时看到输出。 |
| `docker login <url>`➥ `--username <username>`➥ `--password <password>` | 使用您的私有 Docker 仓库进行身份验证，以便您可以对其运行其他命令。 |
| `docker tag <existing-tag>`➥ `<new-tag>` | 为现有镜像添加新标签。要将镜像推送到您的私有容器仓库，您必须首先使用您仓库的 URL 标记它。 |
| `docker push <tag>` | 将带有适当标签的镜像推送到您的私有 Docker 仓库。镜像应带有您仓库的 URL。 |
| `docker kill <container-id>` | 在本地停止特定容器。 |
| `docker rm <container-id>` | 在本地删除特定容器（必须先停止）。 |
| `docker rmi <image-id>`➥ `--force` | 在本地删除特定镜像（必须先删除任何容器）。`--force` 参数即使在镜像被多次标记的情况下也会删除镜像。 |

## B.4 Docker Compose 命令

参见第四章和第五章，了解如何使用 Docker Compose 在您的开发工作站（或个人电脑）上模拟微服务应用程序进行开发和测试。

表 B.3 Docker Compose 命令

| 命令 | 描述 |
| --- | --- |
| `docker-compose --version` | 检查 Docker Compose 是否已安装并打印版本号。 |
| `docker-compose up --build` | 根据当前工作目录中定义的 Docker Compose 文件（docker-compose .yaml）构建并实例化由多个容器组成的应用程序。 |
| `docker-compose ps` | 列出由 Docker Compose 文件指定的应用程序中的运行容器。 |
| `docker-compose stop` | 停止应用程序中的所有容器，但保留已停止的容器以供检查。 |
| `docker-compose down` | 停止并销毁应用程序，使开发工作站处于干净状态。 |

## Terraform 命令

参见第六章和第七章，了解如何通过 Terraform 实现基础设施即代码，以创建 Kubernetes 集群并将您的微服务部署到其中。

表 B.4 Terraform 命令

| 命令 | 描述 |
| --- | --- |
| `terraform init` | 初始化 Terraform 项目并下载提供者插件。 |
| `terraform apply`➥ `-auto-approve` | 在工作目录中执行 Terraform 代码文件，以增量方式应用更改到您的基础设施。 |
| `terraform destroy` | 销毁 Terraform 项目创建的所有基础设施。 |

## B.5 测试命令

查阅第八章了解使用 Jest 和 Cypress 对微服务进行自动化测试的内容。

表 B.5 测试命令

| 命令 | 描述 |
| --- | --- |
| `npx jest --init` | 初始化 Jest 配置文件。 |
| `npx jest` | 在 Jest 下运行测试 |
| `npx jest --watch` | 在启用实时重新加载的情况下运行测试，当代码更改时重新运行测试。Jest 使用 Git 来了解哪些文件已更改，应予以考虑。 |
| `npx jest --watchAll` | 如前所述，但它监视所有文件以更改，而不仅仅是 Git 报告已更改的文件。 |
| `npx cypress open` | 打开 Cypress UI，以便您可以运行测试。实时重新加载默认启用。您可以更新您的代码，测试将自动重新运行。 |
| `npx cypress run` | 在无头模式下运行 Cypress 测试。这允许您从命令行（或 CD 管道）测试 Cypress，而无需显示 UI。 |
| `npm test` | 运行测试的 npm 脚本约定。这可以运行 Jest 或 Cypress（甚至两者都可以），具体取决于您如何配置您的 package.json 文件。这是您应该在您的持续集成（CD）管道中运行的命令，以执行测试套件。 |
| `npm run test:watch` | 这是我在实时重新加载模式下运行测试的个人约定。您需要配置此脚本在您的 package.json 文件中才能使用它。 |
