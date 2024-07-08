# 第七章：图表存储库

没有一种软件包管理器能够完整而不具备发布和分发软件包的方式。组织和供应商必须有一种方法来发布供最终用户下载和使用的软件包。同样，最终用户必须有一种从多种来源获取软件包的共同方式。

Helm 通过称为图表存储库的系统实现软件包分发。图表存储库是简单的 HTTP(S) Web 服务，用户可以从中发现和下载可用的图表。从概念上讲，图表存储库在设计上类似于 Debian 软件包存储库、Fedora 软件包数据库或 Comprehensive Perl Archive Network (CPAN)。

在本章中，我们将首先深入探讨图表存储库的内部工作原理。我们将讨论存储库索引及如何使用新的图表版本更新它。之后，我们将展示如何从头开始设置图表存储库，如何确保其安全，并展示一个使用 GitHub Pages 为开源项目托管公共图表存储库的实际示例。之后，我们将详细介绍各种`helm repo`命令及其有效使用方法。

在本章的末尾，我们将介绍使用 Helm 实验性 Open Container Initiative（OCI）支持的下一代图表存储库。这一前沿功能是在 Helm 3 中添加的，允许用户将 Helm 图表存储在容器注册表中，与其容器镜像并存。

最后，我们将简要描述与图表存储库相关的 Helm 生态系统中的一些项目。

# 存储库索引

所有的图表存储库都包含一个名为*index.yaml*的特殊存储库索引文件，列出了所有可用的图表及其各自的下载位置。

###### 注意

详细描述*index.yaml*格式的更多细节，请参见附录 B。

这里是一个非常基本的*index.yaml*文件示例：

```
apiVersion: v1
entries:
  superapp:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2020-04-27T17:46:52.60919-05:00"
    description: A Helm chart for Kubernetes
    digest: cd1f8d949aeb6a7a3c6720bfe71688d4add794881b78ad9715017581f7867db4
    name: superapp
    type: application
    urls:
    - superapp-0.1.0.tgz
    version: 0.1.0
generated: "2020-04-27T17:46:52.607943-05:00"
```

注意`entries`部分，其中列出了所有图表及其版本。此*index.yaml*示例仅列出了一个图表*superapp*，具有一个版本 0.1.0。

## 图表存储库索引示例

通常，图表存储库会列出许多图表及其所有可用的版本。以下是一个更真实的图表存储库索引示例，包含多个图表及其版本：

```
apiVersion: v1
entries:
  cert-manager:
  - apiVersion: v1
    appVersion: v0.14.2
    created: "2020-04-08T11:38:26.281Z"
    description: A Helm chart for cert-manager
    digest: 160e1bd4906855b91c8ba42afe10af2d0443b184916e4534175890b1a7278f4e
    home: https://github.com/jetstack/cert-manager
    icon: https://raw.githubusercontent.com/jetstack/cert-manager/master/logo/
          logo.png
    keywords:
    - cert-manager
    - kube-lego
    - letsencrypt
    - tls
    maintainers:
    - email: dev@jetstack.io
      name: jetstack-dev
    name: cert-manager
    sources:
    - https://github.com/jetstack/cert-manager
    urls:
    - charts/cert-manager-v0.14.2.tgz
    version: v0.14.2
  - apiVersion: v1
    appVersion: v0.14.1
    created: "2020-03-25T18:30:16.354Z"
    description: A Helm chart for cert-manager
    digest: 629150400487df41af6c7acf2a3bfd8e691f657a930bc81e1dcf3b9d23329baf
    home: https://github.com/jetstack/cert-manager
    icon: https://raw.githubusercontent.com/jetstack/cert-manager/master/logo/
          logo.png
    keywords:
    - cert-manager
    - kube-lego
    - letsencrypt
    - tls
    maintainers:
    - email: dev@jetstack.io
      name: jetstack-dev
    name: cert-manager
    sources:
    - https://github.com/jetstack/cert-manager
    urls:
    - charts/cert-manager-v0.14.1.tgz
    version: v0.14.1
  tor-proxy:
  - apiVersion: v1
    created: "2018-11-16T09:23:13.538Z"
    description: A Helm chart for Kubernetes
    digest: 1d2fd11e22ba58bf0a263c39777f0f18855368b099aed7b03123ca91e55343e4
    name: tor-proxy
    urls:
    - charts/tor-proxy-0.1.1.tgz
    version: 0.1.1
generated: "2020-04-23T17:43:41Z"
```

上述示例显示了两个可用的图表：*cert-manager*和*tor-proxy*。共有三个可用的图表版本：*cert-manager* v0.14.1、*cert-manager* v0.14.2（最新版本）和*tor-proxy* 0.1.1（最新版本）。运行`helm search`时将显示存储库中每个图表的最新版本。

通常，图表归档（*.tgz*文件）本身是从与存储库索引相同的位置提供的，但索引也可能链接到完全不同域上的远程位置。以下是一个从*index.yaml*中引用图表归档的片段，位于单独域上（请注意绝对网址）：

```
...
  - appVersion: 2.10.1
    created: 2019-01-14T23:25:37.125126859Z
    description: A simple, powerful publishing platform that allows you to share
      your stories with the world
    digest: dcadf39f81253a9a016fcab1b74aba1d470e015197152affdaeb1b337221cc5c
    engine: gotpl
    home: http://www.ghost.org/
    icon: https://bitnami.com/assets/stacks/ghost/img/ghost-stack-220x234.png
    keywords:
    - ghost
    - blog
    - http
    - web
    - application
    - nodejs
    - javascript
    maintainers:
    - email: containers@bitnami.com
      name: Bitnami
    name: ghost
    sources:
    - https://github.com/bitnami/bitnami-docker-ghost
    urls:
    - https://charts.example.com/ghost-6.2.3.tgz ![1](img/1.png)
    version: 6.2.3
...
```

![1](img/#co_chart_repositories_CO1-1)

绝对图表 URL

每个条目中还包含在*Chart.yaml*中描述的图表元数据的其他字段，例如`description`，以及包含图表归档的安全哈希算法（SHA-256）摘要字段。在第四章中，我们详细讨论了图表元数据和*Chart.yaml*。

此外，在顶层还有一个`generated`字段，描述索引的创建时间（RFC 3339 格式），以及一个`apiVersion`字段，描述图表存储库的 API 版本。目前只有一个图表存储库的 API 版本。此字段应始终为`v1`。

## 生成索引

存储库索引可以通过自定义程序生成，或者手动输入。Helm 还提供了生成存储库索引的内置功能。

让我们创建一个空目录*charts/*，它将作为我们图表仓库的根目录：

```
$ mkdir -p charts/
```

要在*charts/*目录中生成存储库索引，请运行以下命令：

```
$ helm repo index charts/
```

这将在*charts/index.yaml*创建一个文件。让我们来看一下：

```
$ cat charts/index.yaml
apiVersion: v1
entries: {}
generated: "2020-04-28T09:55:29.517285-05:00"
```

你会注意到`entries`是空的。这是预期的，因为我们在*charts/*目录中还没有任何图表。

让我们创建一个示例图表，并将其打包到*charts/*目录中：

```
$ helm create superapp
Creating superapp
$ helm package superapp/ --destination charts/
Successfully packaged chart and saved it to: charts/superapp-0.1.0.tgz
```

现在让我们再次尝试生成索引：

```
$ helm repo index charts/
$ cat charts/index.yaml
apiVersion: v1
  entries:
    superapp:
    - apiVersion: v2
      appVersion: 1.16.0
      created: "2020-04-28T10:12:22.507943-05:00"
      description: A Helm chart for Kubernetes
      digest: 46f9ddeca12ec0bc257a702dac7d069af018aed2a87314d86b230454ac033672
      name: superapp
      type: application
      urls:
      - superapp-0.1.0.tgz
      version: 0.1.0
generated: "2020-04-28T10:12:22.507289-05:00"
```

现在我们在`entries`部分看到了我们的图表。

## 添加到现有索引

在某些场景下（例如持续集成/持续部署[CI/CD]），你可能只能访问现有的*index.yaml*文件和新打包的图表归档。Helm 提供了一个使用`--merge`选项构建现有索引内容的机制。

让我们模拟这种情况。创建一个名为*workspace/*的新目录，它将代表 CI/CD 管道中的新工作目录：

```
$ mkdir -p workspace/
```

将现有的索引文件复制到*workspace/*目录，并使用新名称，比如*index-old.yaml*：

```
$ cp charts/index.yaml workspace/index-old.yaml
```

在实际场景中，您可能从某个远程位置（例如 Amazon S3）获取现有的索引文件。

接下来让我们创建另一个 Helm 图表，并将其打包到*workspace/*目录中：

```
$ helm create duperapp
Creating duperapp
$ helm package duperapp/ --destination workspace/
Successfully packaged chart and saved it to: workspace/duperapp-0.1.0.tgz
```

运行以下命令，将根据*index-old.yaml*中现有条目的组合以及*workspace/*目录中的任何*.tgz*文件创建一个新的*index.yaml*文件：

```
$ helm repo index workspace/ --merge workspace/index-old.yaml
```

最后，将*workspace/*目录中的文件移动到*charts/*目录中，覆盖旧的索引文件：

```
$ mv workspace/duperapp-0.1.0.tgz charts/
$ mv workspace/index.yaml charts/
```

现在索引文件的新版本应该包含两个图表的条目：

```
$ cat charts/index.yaml
apiVersion: v1
entries:
  duperapp:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2020-04-28T11:34:26.780267-05:00"
    description: A Helm chart for Kubernetes
    digest: 30ea14a4ce92e0d1aea7626cb30dfbac68a87dca360d0d76a55460b004d62f52
    name: duperapp
    type: application
    urls:
    - duperapp-0.1.0.tgz
    version: 0.1.0
  superapp:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2020-04-28T10:12:22.507943-05:00"
    description: A Helm chart for Kubernetes
    digest: 46f9ddeca12ec0bc257a702dac7d069af018aed2a87314d86b230454ac033672
    name: superapp
    type: application
    urls:
    - superapp-0.1.0.tgz
    version: 0.1.0
generated: "2020-04-28T11:34:26.779758-05:00"
```

在不一定具有包含所有图表存档的目录访问权限的环境中，此方法非常有用。

请记住，如果此合并同时在多个系统上发生，您可能会遇到一个竞争条件，其中一个或多个图表可能会从索引中消失。可以通过确保此过程仅同步执行（例如，单个 CI 作业负责为存储库创建*index.yaml*）来减轻此问题。解决此问题的另一种方法是使用负责生成*index.yaml*内容的动态 web 服务器。本章稍后描述的*ChartMuseum*项目在此目的上是一个这样的例子，它是一个动态图表存储库服务器。

# 设置图表存储库

图表存储库的一个好处是它们可以完全静态化——这意味着您可以将文件放在诸如 Apache 或 Nginx 之类的简单 web 服务器后面，并按原样提供服务。您甚至可以使用对象存储提供者，例如 Amazon S3。例如，当客户端请求*index.yaml*时，服务器端不需要进行任何重要的计算。静态 web 服务器只需打开文件并将原始内容发送回客户端。

## 使用 Python 的简单图表存储库

为了本例，我们将使用 Python 的内置静态 web 服务器启动一个本地测试存储库。请注意，几乎所有编程语言的标准库都支持启动 web 服务器并提供静态文件的服务。选择 Python 仅仅是因为它预装在大多数基于 Unix 的系统上，并且因为它提供了一个简单的一行命令来启动静态 web 服务器。

按照上一节的说明（“生成索引”）创建*charts/*目录，其中包含文件*index.yaml*，*superapp-0.1.0.tgz*和*duperapp-0.1.0.tgz*。运行以下命令中的一个以在*[*http://localhost:8080*](http://localhost:8080)*启动本地 web 服务器。

使用 Python 3（首先尝试这个）：

```
$ ( cd charts/ && python3 -m http.server --bind 127.0.0.1 8080 )
```

使用 Python 2：

```
$ ( ch charts/ && python -m SimpleHTTPServer 8080 )
```

###### 小心

这个命令的 Python 2 版本监听所有接口（`0.0.0.0`），而不仅仅是环回接口（`127.0.0.1`）。根据您的系统，这将允许网络中的其他设备连接。在运行此命令之前，请注意*charts/*目录中存在哪些文件。

现在，在另一个终端窗口中，尝试使用`curl`获取*index.yaml*：

```
$ curl -sO http://localhost:8080/index.yaml
$ ls *.yaml
index.yaml
```

现在让我们验证一下我们是否可以获取图表存档：

```
$ curl -sO http://localhost:8080/superapp-0.1.0.tgz
$ curl -sO http://localhost:8080/duperapp-0.1.0.tgz
$ ls *.tgz
duperapp-0.1.0.tgz	superapp-0.1.0.tgz
```

如果`curl`命令成功，您的图表存储库已准备好与 Helm 一起使用。

## 保护图表存储库

在许多情况下，您可能希望限制对图表存储库的访问，或者保持哪些用户访问了哪些资源的审计跟踪。Helm 内置支持，允许用户对由基本身份验证或 mTLS 保护的图表存储库进行身份验证。

### 基本身份验证

图表存储库可以通过基本访问认证或*基本认证*进行保护。这需要用户提供有效的用户名和密码组合以访问服务器上的资源。

服务器可以通过在处理请求之前首先检查`Authorization`头部来实现基本认证。传入的基本认证头部类似于以下内容：

```
Authorization: Basic bXl1c2VyOm15cGFzcw== ![1](img/1.png)
```

![1](img/#co_chart_repositories_CO2-1)

此处的不透明字符串是*`username`* + “:” + *`password`* 的 Base64 编码。

###### 注意

`Authorization`头部的内容*不*加密，因此强烈建议在提供基本认证凭据时也使用 HTTPS。

当首次添加存储库时，您可以在命令行上提供用户名和密码组合，这将指示 Helm 在与该存储库进行请求时使用基本认证：

```
$ helm repo add mycharts http://localhost:8080 --username myuser \
    --password mypass
"mycharts" has been added to your repositories
```

### 客户端证书

大多数通过 HTTPS 进行的客户端-服务器通信允许客户端根据服务器提供的 SSL 证书验证服务器的身份。通过双向 TLS 认证（mTLS），服务器还可以根据客户端在 TLS 握手期间提供的单独 SSL 证书验证客户端的身份。

这是一个简单的 Nginx 服务器配置，用于启用图表存储库的 mTLS，假设静态文件（即*index.yaml*，*.tgz*文件）位于服务器上的目录*/chartrepo*中：

```
events { }
http {
    server {
        root /chartrepo;
        listen 443 ssl;
        server_name charts.example.com;
        ssl_certificate /certs/server.crt; ![1](img/1.png)
        ssl_certificate_key /certs/server.key; ![2](img/2.png)
        ssl_client_certificate /certs/client-ca.pem; ![3](img/3.png)
        ssl_verify_client on;
        proxy_set_header SSL_CLIENT_CERT $ssl_client_cert;
    }
}
```

![1](img/#co_chart_repositories_CO3-1)

服务器的 SSL 证书

![2](img/#co_chart_repositories_CO3-2)

服务器的私钥

![3](img/#co_chart_repositories_CO3-3)

客户端身份验证的证书颁发机构（CA）——只有由此 CA 签名的证书的客户端请求将被接受

获得客户端证书的第一步是生成新的私钥和证书签名请求（CSR）：

```
$ mkdir -p ~/client-certs/
$ cd ~/client-certs/
$ openssl genrsa -out francis.key 4096
$ openssl req -new -key francis.key -out francis.csr
```

在生成证书签名请求（CSR）时，当提示“通用名称”时，您必须输入一个值。使用能够识别客户端的内容（例如“francis”）。其他字段在技术上可以留空，尽管鼓励您填写它们。

接下来，使用服务器上配置的证书颁发机构（`client-ca.pem`）和相关私钥（`client-ca.key`），从 CSR 生成新的客户端证书：

```
$ openssl x509 -req -in francis.csr \
    -CA /certs/client-ca.pem -CAkey /certs/client-ca.key \
    -out francis.crt -sha256
```

现在您可以通过在添加新图表存储库时指定`--cert-file`和`--key-file`选项来使用此证书进行身份验证：

```
$ helm repo add client-cert-repo https://charts.example.com \
    --cert-file ~/client-certs/francis.crt --key-file ~/client-certs/francis.key
"client-cert-repo" has been added to your repositories
```

如果您的服务器使用自签名证书，您还可以指定`--ca-file`选项，指向一个受信任的证书或证书包：

```
$ helm repo add client-cert-repo-selfsigned https://charts.example.com \
    --cert-file ~/client-certs/francis.crt --key-file ~/client-certs/francis.key
    --ca-file /certs/server.crt
"client-cert-repo-selfsigned" has been added to your repositories
```

###### 注意

用于`--cert-file`，`--key-file`和`--ca-file`的路径都存储在与存储库相关的 Helm 缓存中。不要移动这些文件，否则将由于客户端需要的缺失文件而导致对存储库的未来请求失败。

有关更多关于 mTLS 的信息，请参阅互联网工程任务组（IETF）RFC 8446，“传输层安全性（TLS）协议版本 1.3”。

## 真实示例：使用 GitHub Pages

GitHub 提供了一个免费的静态托管解决方案，称为 GitHub Pages。如果您不介意将您的图表公开给世界，GitHub Pages 是托管图表存储库的绝佳选择，因为您不需要支付任何费用。

更好的是，GitHub Pages 允许您使用指向您的 GitHub Pages 站点的自定义域名。本节将展示如何使用 GitHub Pages 轻松设置一个公共 Helm 图表存储库。

GitHub Pages 存在一些限制（如带宽），因此在使用此方法之前，请根据您的图表存储库的性能要求与 GitHub Pages 功能文档进行比较。

### 创建一个新的 Git 存储库

第一步是在 GitHub 上创建一个全新的 Git 存储库，专门用于您的图表存储库。您理论上可以将图表存储库与其他内容一起托管，但为了简单起见，我们将使用一个专用的 Git 存储库。图 7-1 显示了如何设置一个新的存储库。

![GitHub Pages](img/lehe_0701.png)

###### 图 7-1\. 在 GitHub 中创建新的公共存储库

一旦您登录 GitHub，点击屏幕右上角，选择“新建存储库”。为 Git 存储库命名任何您喜欢的名称。在本示例中，我们将使用名称 *mycharts*。确保选择将存储库标记为“Public”，这是使用 GitHub Pages 的先决条件。选择“Initialize this repository with a README” 的选项，这将允许我们立即克隆存储库。随意选择许可证，如“MIT License”，表示该存储库中的源代码可自由使用和重新用途。最后，点击“Create repository”。

###### 注意

在这个背景下，需要注意 Helm 存储库（或图表存储库）与托管在 GitHub 上的 Git 存储库之间的区别，后者用于版本控制。

### 启用 GitHub Pages

转到存储库的设置面板。在主设置中，向下滚动到名为 GitHub Pages 的部分（参见 图 7-2）。对于 Source 选项，请选择“main branch”。这将导致 GitHub 在您提交新的提交到主分支时重新部署您的 GitHub Pages 站点。点击保存。

![GitHub Pages](img/lehe_0702.png)

###### 图 7-2\. 在您的存储库上启用 GitHub Pages

### 可选：使用自定义域名

GitHub Pages 上的站点默认作为 `github.io` 域的子域托管。例如，您的站点的 URL 将类似于 *https://`yourusername`.github.io/mycharts/*。

如果你有一个要使用的自定义域名，在你的注册商的网页控制台（或者替代方案，你设置了用于你授权域名服务器的服务的控制台），创建一个新的 DNS 记录，指向*`yourusername`.github.io*。如果使用根域名，使用`ALIAS`记录类型；否则，对于子域名，使用`CNAME`记录类型。

返回 GitHub 中的仓库设置。如图 7-3，在“自定义域名”输入框中，输入你为其设置了 DNS 记录的域名。

![GitHub Pages, 500](img/lehe_0703.png)

###### 图 7-3\. 使用 GitHub Pages 的自定义域名

GitHub 可能需要最多一个小时来为你的域生成 TLS 证书。一旦准备好，你应该在设置中看到一些文本显示“你的站点发布在 https://example.com。”一旦看到这个，确保启用“强制 HTTPS”选项，这样你的站点只能通过 HTTPS 访问，而不是普通的 HTTP。

### 添加图表仓库文件

在 GitHub UI 中找到你仓库的克隆 URL（通常在屏幕右侧）。将你的新 GitHub 仓库克隆到本地系统，以便我们添加一些文件将其转换成一个真正的图表仓库：

```
$ git clone git@github.com:youruser/mycharts.git
Cloning into 'mycharts'...
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 7 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (7/7), done.
```

输入你的 Git 仓库的目录：

```
$ cd mycharts/
```

接下来，让我们在一个新的*src/*目录中创建一个名为*pineapple*的图表，将其打包成一个存档文件放在仓库的根目录，并创建一个*index.yaml*文件：

```
$ mkdir -p src/
$ helm create src/pineapple
Creating src/pineapple
$ helm package src/pineapple/
Successfully packaged chart and saved it to: /home/user/mycharts/
  pineapple-0.1.0.tgz
$ helm repo index .
```

完成后，让我们把所有这些新文件都提交并推送回 GitHub：

```
$ git add .
```

```
$ git commit -m "Add pineapple chart v0.1.0"
[main 9bba19d] Add pineapple chart v0.1.0
 13 files changed, 395 insertions(+)
 create mode 100644 index.yaml
 create mode 100644 pineapple-0.1.0.tgz
 create mode 100644 src/pineapple/.helmignore
 create mode 100644 src/pineapple/Chart.yaml
 create mode 100644 src/pineapple/templates/NOTES.txt
 create mode 100644 src/pineapple/templates/_helpers.tpl
 create mode 100644 src/pineapple/templates/deployment.yaml
 create mode 100644 src/pineapple/templates/hpa.yaml
 create mode 100644 src/pineapple/templates/ingress.yaml
 create mode 100644 src/pineapple/templates/service.yaml
 create mode 100644 src/pineapple/templates/serviceaccount.yaml
 create mode 100644 src/pineapple/templates/tests/test-connection.yaml
 create mode 100644 src/pineapple/values.yaml
```

```
$ git push origin main
Enumerating objects: 20, done.
Counting objects: 100% (20/20), done.
Delta compression using up to 12 threads
Compressing objects: 100% (17/17), done.
Writing objects: 100% (19/19), 9.29 KiB | 4.64 MiB/s, done.
Total 19 (delta 0), reused 0 (delta 0)
To github.com:youruser/mycharts.git
   4964b76..9bba19d  main -> main
```

返回浏览器中的 GitHub。在你推送改变和这些改变在 GitHub Pages 网站上可用之间会有一小段延迟。点击右侧边栏中的“环境”项。这会告诉你你的站点上一次部署的时间。如果你看到引用了你刚刚推送的提交（在前面的例子中是`9bba19d`），你的 GitHub Pages 站点已经可以使用了。

### 将你的 GitHub Pages 站点用作图表仓库

一旦你上传了一个*index.yaml*文件到你的 Git 仓库，并且站点已经使用最新的提交上线，你可以像使用任何其他图表仓库一样使用它。

将你的 GitHub Pages 图表仓库添加到你的本地仓库：

```
$ helm repo add gh-pages https://yourusername.github.io/mycharts/
```

或者，如果你正在使用自定义域名：

```
$ helm repo add gh-pages https://example.com
```

# 使用图表仓库

一旦你有了一个工作的图表仓库（参见前面的章节），你可以使用 Helm CLI 来利用它。

顶级`helm repo`子命令下提供了几个命令来处理图表仓库。本节将重点介绍如何有效地使用每个命令。

## 添加一个仓库

使用图表仓库的第一步是给它分配一个唯一的名称（如`mycharts`），并将其添加到 Helm 已知的仓库列表中。当你第一次添加一个仓库时，Helm 会从提供的 URL 获取*index.yaml*并将其存储在本地。

使用`helm repo add`命令添加你的仓库：

```
$ helm repo add mycharts http://localhost:8080
"mycharts" has been added to your repositories
```

如果您正在运行 Python 示例，请查看您的图表存储库日志，您应该会看到有关`GET /index.yaml`的传入请求：

```
127.0.0.1 - - [06/May/2020 15:31:07] "GET /index.yaml HTTP/1.1" 200 -
```

## 下载图表

要从存储库直接下载图表，请使用`helm pull`命令：

```
$ helm pull mycharts/superapp
```

根据语义版本，Helm 将自动找到最新版本。您还可以指定一个版本：

```
$ helm pull mycharts/superapp --version 0.1.0
```

这将在您的工作空间中生成一个新的图表存档（*.tgz*文件）：

```
$ ls *.tgz
superapp-0.1.0.tgz
```

然后可以直接安装此存档：

```
$ helm install superapp-dev superapp-0.1.0.tgz
```

您还可以直接从添加的存储库安装图表：

```
$ helm install superapp-dev mycharts/superapp
```

## 列出存储库

知道已经添加到您系统中的哪些图表存储库通常是有帮助的。这可能会帮助您决定是否要使用其中一个来下载图表，或者彻底从系统中移除其中一个。

使用`helm repo list`命令列出添加到您系统的所有图表存储库：

```
$ helm repo list
NAME    	URL
mycharts	http://localhost:8080
```

如果需要，您还可以利用`--output` / `-o`选项以机器可读格式获取这些内容。

通过添加`-o yaml`来获取 YAML 格式的列表：

```
$ helm repo list -o yaml
- name: mycharts
  url: http://localhost:8080
```

通过添加`-o json`来获取 JSON 格式的列表：

```
$ helm repo list -o json
[{"name":"mycharts","url":"http://localhost:8080"}]
```

## 更新存储库

一旦发布新的图表版本，存储库所有者就会将*.tgz*包添加到存储库存储中，并使用新条目更新*index.yaml*。

为了获取存储库索引的最新版本，请使用`helm repo update`命令：

```
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "mycharts" chart repository
Update Complete. Happy Helming!
```

如果您正在运行 Python 示例，则应该注意到图表存储库的输出日志中会有一个关于`GET /index.yaml`的传入请求：

```
127.0.0.1 - - [06/May/2020 15:31:07] "GET /index.yaml HTTP/1.1" 200 -
```

无论存储库索引是否已更改内容（我们没有向`myrepo`添加更多图表），该文件都会被获取并下载到缓存中，覆盖先前保存的版本。

## 删除存储库

为了移除存储库，您可以使用`helm repo remove`：

```
$ helm repo remove mycharts
"mycharts" has been removed from your repositories
```

这将删除 Helm 缓存中存储的有关此存储库的所有引用。

# 实验性 OCI 支持

###### 警告

[Helm 的 OCI 支持](https://oreil.ly/eH4KE)仍然被认为是高度实验性的。尽管这一领域的开发仍在积极进行，但本节描述的语法可能很快就会过时。

图表存储库系统旨在易于使用。在大多数情况下，这个系统已被证明是足够的——使全球各地的组织能够共享和分发他们的 Helm 图表。

然而，图表存储库确实存在一些关键挑战：

+   它们没有命名空间的概念；存储库中的所有图表都列在单个索引中。

+   它们没有细粒度的访问控制；您要么可以访问存储库中的所有图表，要么不能访问任何图表。

+   图表包使用不同名称但完全相同的原始内容存储两次。

+   存储库索引可能变得非常庞大，导致 Helm 消耗大量内存。

与尝试添加功能以解决当前图表存储库模型的所有这些问题不同，将下一代图表存储库构建在符合*OCI 分发规范*的注册表上是更为合理的选择。

OCI 代表 *Open Container Initiative*。从 [*https://opencontainers.org*](https://opencontainers.org) 网站上得知，OCI 的定义如下：

> 一个开放的治理结构，旨在围绕容器格式和运行时创建开放行业标准。

OCI 定义的标准之一是 *分发规范*。该规范描述了用于分发容器镜像的 HTTP API。有趣的是，此 API 是通用的，并且适用于不是容器镜像的各种内容，比如 Helm 图表！

从 Helm 3.0.0 开始，添加了实验性支持，可以推送和拉取基于 OCI 的容器注册表中的图表。

## 启用 OCI 支持

目前为止，Helm 的 OCI 支持仍然被视为实验性质。

目前，请在您的环境中设置以下内容以启用 OCI 支持：

```
$ export HELM_EXPERIMENTAL_OCI=1
```

## 运行本地注册表

Docker Distribution 项目（也称为 Docker 注册表）是 Docker 的 Registry v2 API 的原始实现。它原生支持 Helm 图表。

如果已安装 `docker`，可以使用以下命令在端口 5000 上的容器中轻松运行本地注册表：

```
$ docker run -d --name oci-registry -p 5000:5000 registry
```

要查看注册表的日志，请运行以下命令（按 Ctrl-C 退出）：

```
$ docker logs -f oci-registry
```

要停止注册表，请运行以下命令：

```
$ docker rm -f oci-registry
```

Docker 注册表有关于认证、存储等方面的 [多个配置选项](https://oreil.ly/CBb0F)。

如果希望使用单个用户名密码组合配置基本认证，首先创建一个 *.htpasswd* 文件：

```
$ htpasswd -cB -b auth.htpasswd myuser mypass
```

然后启动注册表，挂载 *.htpasswd* 文件并设置 `REGISTRY_AUTH` 环境变量：

```
$ docker run -d --name oci-registry -p 5000:5000 \
    -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
    -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry \
    auth.htpasswd}}" registry
```

有关 Docker Distribution 的更多信息，请访问 [项目 GitHub 页面](https://oreil.ly/Q8Omf)。

## 登录到注册表

要对注册表进行身份验证，请使用 `helm registry login` 命令（系统会提示您手动输入密码）：

```
$ helm registry login -u myuser localhost:5000
Password:
Login succeeded
```

它会使用凭据在注册表上的路径 `/v2/` 发出简单的 `GET` 请求，以确定它们是否有效。如果有效，凭据将存储在 Helm 配置文件中。如果启用了任何 Docker 凭据存储（例如 macOS 上的 `osxkeychain`），用户名和密码将安全地存储在那里。

###### 注意

在 `localhost:5000` 运行本地注册表的示例未使用认证。如果尚未在注册表上启用认证，则将接受任何登录凭据的组合。

## 从注册表注销

要从系统中删除特定注册表的凭据，请使用 `helm registry logout` 命令：

```
$ helm registry logout localhost:5000
Logout succeeded
```

## 将图表存储到缓存中

在将图表上传到注册表之前，必须先将其保存到缓存中。这将把图表从其正常状态转换为内容可寻址的块，并为其提供唯一标识符。

使用 `helm chart save` 将图表存储到缓存中：

```
$ helm chart save mychart/ localhost:5000/myrepo/mychart
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 2.7.0
2.7.0: saved
```

注意，在图表引用中使用的标签基于 *Chart.yaml* 中的版本（`2.7.0`）。

您还可以通过在图表引用后的冒号（`:`）之后指定自定义标签，如 `stable`，来使用自定义标签：

```
$ helm chart save mychart/ localhost:5000/myrepo/mychart:stable
ref:     localhost:5000/myrepo/mychart:stable
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 2.7.0
stable: saved
```

## 在缓存中列出图表

使用 `helm chart list` 来显示当前存储在缓存中的所有图表：

```
$ helm chart list
REF                                               VERSION DIGEST  SIZE
localhost:5000/myrepo/mychart:2.7.0               2.7.0   84059d7 454 B
localhost:5000/stable/acs-engine-autoscaler:2.2.2 2.2.2   d8d6762 4.3 KiB
localhost:5000/stable/aerospike:0.2.1             0.2.1   4aff638 3.7 KiB
localhost:5000/stable/airflow:0.13.0              0.13.0  c46cc43 28.1 KiB
localhost:5000/stable/anchore-engine:0.10.0       0.10.0  3f3dcd7 34.3 KiB
```

## 从缓存中导出图表

如果您希望在缓存中存在的图表中提取源文件，必须先将其导出到本地目录。使用 `helm chart export` 命令来导出图表：

```
$ helm chart export localhost:5000/myrepo/mychart:2.7.0
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 2.7.0
Exported chart to mychart/
```

图表的名称将用于确定输出路径（例如，`mychart/`）。

## 将图表推送到注册表

将图表推送（即上传）到注册表允许其他人使用它。一旦您已经登录到注册表，并且要推送的图表已保存到缓存中，请使用 `helm chart push` 命令来推送图表：

```
$ helm chart push localhost:5000/myrepo/mychart:2.7.0
The push refers to repository [localhost:5000/myrepo/mychart]
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 2.7.0
2.7.0: pushed to remote (1 layer, 2.4 KiB total)
```

## 从注册表中拉取图表

一旦图表被推送到注册表，其他用户就可以拉取（即下载）它们。从注册表拉取图表会将它们放入本地缓存中。要从注册表拉取现有的图表，请使用 `helm chart pull` 命令：

```
$ helm chart pull localhost:5000/myrepo/mychart:2.7.0
2.7.0: Pulling from localhost:5000/myrepo/mychart
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 2.7.0
Status: Downloaded newer chart for localhost:5000/myrepo/mychart:2.7.0
```

## 从缓存中移除图表

要从本地缓存中移除图表，请使用 `helm chart remove` 命令：

```
$ helm chart remove localhost:5000/myrepo/mychart:2.7.0
2.7.0: removed
```

# 相关项目

Helm 的图表存储库系统已经衍生出一系列开源工具，进一步增强了这一体验。以下小节介绍了与图表存储库相关的一些项目。

## ChartMuseum

*项目主页：[*https://github.com/helm/chartmuseum*](https://github.com/helm/chartmuseum)*

*ChartMuseum* 是一个简单的图表存储库 Web 服务器。配置它指向包含图表包的存储位置，它会动态生成 *index.yaml*。它还提供了一个 HTTP API，用于从存储中上传、查询和删除图表包。此外，它还具有用于身份验证、多租户和缓存的多个其他配置设置，使其成为托管私有或内部图表存储库的热门选择。

## Harbor

*项目主页：[*https://github.com/goharbor/harbor*](https://github.com/goharbor/harbor)*

*Harbor* 是一个功能齐全的注册表，具有增强的安全性和管理功能。它提供了一个用于 Helm 图表的 UI，并在后端利用 ChartMuseum 作为多租户图表存储库。它还支持 Helm 的实验性 OCI 功能集。

类似于 Helm，*Harbor* 是一个毕业生，顶级 CNCF 项目。

## Chart Releaser

*项目主页：[*https://github.com/helm/chart-releaser*](https://github.com/helm/chart-releaser)*

*Chart Releaser*，或者 `cr`，是一个命令行工具，利用 GitHub 发布来托管图表包。它能够检测 Git 仓库中的图表，打包它们，并将每个版本命名为唯一的图表版本后上传到 GitHub 发布的工件。

使用 `cr` 上传图表后，该工具还可以基于 GitHub 发布和相关工件的内容生成一个 *index.yaml* 文件。这个仓库索引可以静态托管在 GitHub Pages 或其他地方。

## S3 插件

*项目主页：[*https://github.com/hypnoglow/helm-s3*](https://github.com/hypnoglow/helm-s3)*

S3 插件是 Helm 的一个插件，允许你将私有 Amazon S3 存储桶作为一个图表仓库使用。

## GCS 插件

*项目主页：[*https://github.com/hayorov/helm-gcs*](https://github.com/hayorov/helm-gcs)*

GCS 插件是 Helm 的一个插件，允许你将私有 Google Cloud 存储桶作为一个图表仓库使用。

## Git 插件

*项目主页：[*https://github.com/aslafy-z/helm-git*](https://github.com/aslafy-z/helm-git)*

Git 插件是 Helm 的一个插件，允许你将包含图表源文件的 Git 仓库作为一个图表仓库使用。它支持子路径、自定义引用和 HTTPS 和 SSH Git URL。
