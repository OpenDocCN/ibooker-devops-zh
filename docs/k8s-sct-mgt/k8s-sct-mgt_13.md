# 附录 B. 安装和配置 yq

大多数操作系统都内置了对终端内文本操作的支持，例如 sed 和 awk。这些工具非常适合进行相对简单的文本操作，但在处理结构化序列化语言（如 YAML）时可能会变得繁琐。`yq` ([`mikefarah.gitbook.io/yq`](https://mikefarah.gitbook.io/yq)) 是一个命令行工具，它提供了查询和操作基于 YAML 内容的支持，并且在与 Kubernetes 环境交互时非常有用，因为大部分资源都是以 YAML 格式表达的。另一个类似且流行的工具，称为 `jq`，为 JSON 格式的内容提供了类似的功能，这在第四章中有描述。本附录描述了 `yq` 的安装，以及一些示例来确认工具安装成功。

## B.1 安装 yq

`yq` 支持多个操作系统，可以使用多种方法进行安装，包括包管理器或直接从项目网站 ([`github.com/mikefarah/yq/releases/latest`](https://github.com/mikefarah/yq/releases/latest)) 下载二进制文件。直接二进制选项是最直接的方法，因为它没有外部依赖或先决条件。请确保找到版本 4 或更高版本的工具，因为与先前版本相比，它进行了重大重写。发布页面将允许您选择下载压缩归档或直接二进制文件。或者，您可以使用终端将二进制文件下载到本地机器。

警告：还有一个工具，也称为 `yq`，它作为一个 Python 包提供，并具有类似的功能。安装错误的工具会导致错误，因为这两个工具在语法和功能上存在差异。

可以使用以下命令下载 `yq` 二进制文件并将其放置在 `PATH` 目录中：

列表 B.1 下载 `yq`

```
sudo curl -o /usr/bin/yq
➥https://github.com/mikefarah/yq/releases/download/${VERSION}/${BINARY}  ①
➥&& \ 
    sudo chmod +x /usr/bin/yq
```

① 版本（VERSION）指的是标记的发布版本，而二进制（BINARY）是 yq 二进制文件名称、操作系统名称和架构的组合。例如，AMD64 Linux 的示例是 yq_linux_amd64。

如果以下命令成功执行，则表示安装成功：

```
yq --version
```

如果命令没有成功执行，请确认文件已成功下载，放置在操作系统的 `PATH` 目录中，并且二进制文件是可执行的。

## B.2 以示例说明 yq

`yq` 可以对 YAML 格式的文件执行操作，例如查询和操作值，当与 Kubernetes 清单一起工作时非常有用。`kubectl` 具有多种输出格式的功能，例如 JSONPath 或 Go 模板。然而，某些高级功能，如管道，不可用，需要更全面的专用工具。

为了展示 `yq` 可以用来操作 YAML 内容的一些方法，创建一个名为 `book-info.yaml` 的文件，包含我们熟悉的一个资源：Kubernetes 密钥。

列表 B.2 book-info.yaml

```
apiVersion: v1
metadata:
  name: book-info
stringData:
  title: Securing Kubernetes Secrets
  publisher: Manning
type: Opaque
kind: Secret
```

`yq`有两个主要模式：评估单个文档（使用`evaluate`或`e`子命令）或多个文档（使用`eval-all`或`ea`子命令）。评估单个文档是最常见的模式，因此可以使用它来查询列表 B.2 中创建的`book-info.yaml`文件的 内容。

对 YAML 文件的操作使用*表达式*来确定要执行的具体操作。由于大多数 YAML 内容使用嵌套内容，这些属性可以通过点符号访问。例如，为了提取`stringData`下`title`字段的值，使用表达式`.stringData.title`。在`yq`中使用时，需要三个组件：要执行的操作类型（如`evaluate`）、表达式和 YAML 内容的定位。现在使用以下命令提取`title`字段：

```
yq eval '.stringData.title' book-info.yaml
```

更复杂的表达式也可以用来执行高级操作。*管道*（`|`）可以用来连接表达式，使得一个表达式的输出成为另一个表达式的输入。例如，为了确定`stringData`下`publisher`字段的长度，可以使用管道将`.stringData.publisher`表达式的输出传递给`length`运算符：

```
yq eval '.stringData.publisher | length' book-info.yaml
```

命令的结果应该返回`7`。除了提取属性外，`yq`还可以用来修改 YAML 内容。现在在`stringData`下添加一个名为`category`的新属性，其值为`Security`。完成此任务的`yq`表达式为`'.stringData.category' `=` `"Security"`。执行以下命令以添加`category`字段：

```
yq eval '.stringData.category = "Security"' book-info.yaml
```

以下列表应该是命令的结果。

列表 B.3 使用`yq`更新 YAML 内容的输出

```
apiVersion: v1
metadata:
  name: book-info
stringData:
  title: Securing Kubernetes Secrets
  publisher: Manning
  category: Security     ①
type: Opaque
kind: Secret
```

① 添加了一个新的类别字段。

尽管执行的结果导致添加了新的`category`字段，但重要的是要注意`book-info.yaml`文件的内容并未被修改。为了更新`book-info.yaml`字段的内容，必须使用`-i`选项，这将指示`yq`执行文件就地更新。执行以下命令以执行就地文件更新：

```
yq eval -i '.stringData.category = "Security"' book-info.yaml
```

确认更改已应用于文件。`yq`提供的提取和操作能力使其成为一个多功能的工具，在处理任何 YAML 格式内容时都非常有用。
