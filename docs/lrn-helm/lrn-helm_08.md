# 第八章：Helm 插件和启动器

正如我们在本书中所见，Helm 具有丰富的功能和方法，有助于在 Kubernetes 上交付应用程序。但是，还可以自定义和扩展 Helm 提供的功能。

在本章中，我们将讨论两种进一步增强和自定义您使用 Helm 的方式：*插件*和*启动器*。

插件允许您为 Helm 添加额外功能，并与 CLI 无缝集成，因此对于具有独特工作流需求的用户而言，它们是一个受欢迎的选择。在线上有许多第三方插件可供常见用例使用，如秘密管理。此外，对于独特的一次性任务，构建自己的插件也非常容易。

启动器扩展了使用 `helm create` 生成新的 Helm 图表的可能性。例如，您可能已经为一个内部微服务构建了一个完全符合未来微服务需求的 Helm 图表示例。您可以将该图表转换为一个启动器，然后在启动新项目时每次使用。

通过利用插件和启动器，我们可以构建在 Helm 的开箱即用功能之上，简化和自动化日常工作流任务。

# 插件

Helm 插件是可以直接从 Helm CLI 访问的外部工具。它们允许您向 Helm 添加自定义子命令，而无需对 Helm 的 Go 源代码进行任何修改。这在设计上类似于其他工具中实现插件系统的方式，比如 `kubectl`（Kubernetes CLI）。

此外，下载器插件允许您指定用于与图表仓库通信的自定义协议。如果您有某种自定义身份验证方法，或者需要修改 Helm 从仓库获取图表的方法，这将非常有用。

## 安装第三方插件

许多第三方插件是开源的，并且在 GitHub 上公开可用。其中许多插件使用“helm-plugin”标签/主题，使其易于查找。请参阅 [GitHub 上的 Helm 插件文档](https://oreil.ly/3KwNb)。

安装插件后，获取其版本控制 URL。这将用作获取 *plugin.yaml* 和插件源代码的正确版本的手段。

目前支持 Git、SVN、Bazaar (Bzr) 和 Mercurial (Hg) 的 URL。对于 Git，版本控制 URL 看起来像 *`https://example.com/myorg/myrepo.git`*。

例如，有一个简单的插件用于管理位于 git 仓库中的 Helm 启动器，位于 [*https://github.com/salesforce/helm-starter*](https://github.com/salesforce/helm-starter)。此插件的版本控制 URL 为 `https://github.com/salesforce/helm-starter.git`。

要安装此插件，请运行 `helm plugin install`，将版本控制 URL 作为第一个参数传递：

```
$ helm plugin install https://github.com/salesforce/helm-starter.git
Installed plugin: starter
```

如果安装成功，您可以继续使用该插件：

```
$ helm starter --help
Fetch, list, and delete helm starters from github.

Available Commands:
    helm starter fetch GITURL       Install a bare Helm starter from Github
                                      (e.g., git clone)
    helm starter list               List installed Helm starters
    helm starter delete NAME        Delete an installed Helm starter
    --help                          Display this text
```

要列出所有安装的插件，请使用`helm plugin list`命令：

```
$ helm plugin list
NAME   	VERSION	DESCRIPTION
starter	1.0.0  	This plugin fetches, lists, and deletes helm starters from github.
```

要尝试更新插件，请使用`helm plugin update`命令：

```
$ helm plugin update starter
Updated plugin: starter
```

如果您希望卸载系统中的插件，请使用`helm plugin remove`命令：

```
$ helm plugin remove starter
Uninstalled plugin: starter
```

除非另有规定，Helm 将在安装插件时使用位于 Git 存储库默认分支上的*plugin.yaml*和源代码。如果您希望指定要使用的 Git 标签，请在安装时使用`--version`标志：

```
$ helm plugin install https://github.com/databus23/helm-diff.git --version v3.1.0
```

也可以直接从 tarball URL 安装插件。Helm 将下载 tarball 并解压到插件目录：

```
$ helm plugin install https://example.com/archives/myplugin-0.6.0.tar.gz
```

此外，您还可以从本地目录安装插件：

```
$ helm plugin install /path/to/myplugin
```

Helm 不会复制文件，而是会创建到原始文件的符号链接：

```
$ ls -la "$(helm env HELM_PLUGINS)"
total 8
drwxrwxr-x 2 myuser myuser 4096 Jul  3 21:49 .
drwxrwxr-x 4 myuser myuser 4096 Jul  1 21:38 ..
lrwxrwxrwx 1 myuser myuser   21 Jul  3 21:49 myplugin -> /path/to/myplugin
```

例如，如果您正在积极开发插件，则可能会很有用。在调用符号链接插件时，对*plugin.yaml*和其他源文件进行更改将立即被识别。

## 自定义子命令

插件具有许多有用的功能，可以与现有的 Helm 用户体验无缝集成。Helm 插件最显著的特点可能是每个插件为 Helm 提供一个自定义的顶级子命令。这些子命令甚至可以利用 Shell 完成（本章后面讨论）。

安装插件后，将根据插件名称为您提供一个新的命令可供使用。这个新命令直接集成到 Helm 中，甚至会显示在`helm help`中。

例如，假设我们安装了一个名为`inspect-templates`的插件，它可以为我们提供关于图表中找到的 YAML 模板的额外信息。此插件将为您提供一个额外的 Helm 命令：

```
$ helm inspect-templates [args]
```

这将执行`inspect-templates`插件，将任何参数或标志传递给插件调用时执行的基础工具。插件的作者指定了一些命令，Helm 应在每次调用插件时作为子进程运行（有关如何指定此内容的更多信息，请参阅“构建插件”）。

插件为增强 Helm 现有功能集提供了一个令人满意的替代方案，而无需对 Helm 本身进行任何修改。

## 构建插件

构建 Helm 插件是一个非常简单的过程。根据插件的要求和整体复杂性，可能需要一些编程知识；但是，许多插件仅运行基本的 Shell 命令。

### 底层实现

考虑以下 Bash 脚本，*inspect-templates.sh*，这是我们示例`inspect-templates`插件的底层实现：

```
#!/usr/bin/env bash
set -e

# First argument on the command line, a relative path to a chart directory
CHART_DIRECTORY="${1}"

# Fail if no chart directory provided or is invalid
if [[ "${CHART_DIRECTORY}" == "" ]]; then
    echo "Usage: helm inspect-templates <chart_directory>"
    exit 1
elif [[ ! -d "${CHART_DIRECTORY}" ]]; then
    echo "Invalid chart directory provided: ${CHART_DIRECTORY}"
    exit 1
fi

# Print a summary of the chart's templates
cd "${CHART_DIRECTORY}"
cd templates/
echo "----------------------"
echo "Chart template summary"
echo "----------------------"
echo ""
total="$(find . -type f -name '*.yaml' -maxdepth 1 | wc -l | tr -d '[:space:]')"
echo " Total number: ${total}"
echo ""
echo " List of templates:"
for filename in $(find . -type f -name '*.yaml' -maxdepth 1 | sed 's|^\./||'); do
    kind=$(cat "${filename}" | grep kind: | head -1 | awk '{print $2}')
    echo "  - ${filename} (${kind})"
done
echo ""
```

当用户运行`helm inspect-templates`时，Helm 将在幕后执行此脚本。

###### 注意

插件的底层实现并不需要使用 Bash、Go 或任何特定的编程语言编写。对于此插件的最终用户，它应该看起来只是 Helm CLI 的另一部分。

### 插件清单

每个插件由一个名为*plugin.yaml*的 YAML 文件定义。此文件包含插件元数据和有关在调用插件时运行的命令的信息。

下面是*plugin.yaml*的基本示例，适用于`inspect-templates`插件：

```
name: inspect-templates ![1](img/1.png)
version: 0.1.0 ![2](img/2.png)
description: get a summary of a chart's templates ![3](img/3.png)
command: "${HELM_PLUGIN_DIR}/inspect-templates.sh" ![4](img/4.png)
```

![1](img/#co_helm_plugins_and_starters_CO1-1)

插件的名称。

![2](img/#co_helm_plugins_and_starters_CO1-2)

插件的版本。

![3](img/#co_helm_plugins_and_starters_CO1-3)

插件的基本描述。

![4](img/#co_helm_plugins_and_starters_CO1-4)

调用此插件时运行的命令。

### 手动安装

首先检查插件存储根目录的配置路径：

```
$ HELM_PLUGINS="$(helm env HELM_PLUGINS)"
$ echo "${HELM_PLUGINS}"
/home/myuser/.local/share/helm/plugins
```

# 使用自定义根目录进行插件操作

插件的根目录可以通过在当前环境中提供`HELM_PLUGINS`环境变量的自定义路径进行覆盖。

在插件存储根目录内创建与插件名称（`inspect-templates`）匹配的目录：

```
$ PLUGIN_ROOT="${HELM_PLUGINS}/inspect-templates"
$ mkdir -p "${PLUGIN_ROOT}"
```

接下来，将*plugin.yaml*和*inspect-templates.sh*复制到新目录，并确保脚本可执行：

```
$ cp plugin.yaml "${PLUGIN_ROOT}"
$ cp inspect-templates.sh "${PLUGIN_ROOT}"
$ chmod +x "${PLUGIN_ROOT}/inspect-templates.sh"
```

### 最终结果

下面展示了我们的`inspect-templates`插件的工作示例：

```
$ helm inspect-templates
Usage: helm inspect-templates <chart_directory>
Error: plugin "inspect-templates" exited with error

$ helm inspect-templates nonexistant/
Invalid chart directory provided: nonexistant/
Error: plugin "inspect-templates" exited with error

$ helm create mychart
Creating mychart

$ helm inspect-templates mychart/
----------------------
Chart template summary
----------------------

 Total number: 5

 List of templates:
  - serviceaccount.yaml (ServiceAccount)
  - deployment.yaml (Deployment)
  - service.yaml (Service)
  - hpa.yaml (HorizontalPodAutoscaler)
  - ingress.yaml (Ingress)
```

注意提供的命令行参数（即`mychart/`）直接传递给脚本。这使得插件作者能够构建接受任意数量参数或自定义标志的独立工具变得容易。

## plugin.yaml

*plugin.yaml*是描述插件及其调用命令等重要细节的插件清单文件的名称。

下面是一个包含所有可为插件指定的选项的*plugin.yaml*示例：

```
name: myplugin ![1](img/1.png)
version: 0.3.0 ![2](img/2.png)
usage: "helm myplugin --help" ![3](img/3.png)
description "a plugin that belongs to me" ![4](img/4.png)
platformCommand: ![5](img/5.png)
  - os: windows
    arch: amd64
    command: "bin/myplugin.exe"
command: "bin/myplugin" ![6](img/6.png)
ignoreFlags: false ![7](img/7.png)
hooks: ![8](img/8.png)
  install: "scripts/install-hook.sh"
  update: "scripts/update-hook.sh"
  delete: "scripts/delete-hook.sh"
downloaders: ![9](img/9.png)
  - command: "bin/myplugin-myp-downloader"
    protocols:
      - "myp"
      - "myps"
```

![1](img/#co_helm_plugins_and_starters_CO2-1)

插件的名称。

![2](img/#co_helm_plugins_and_starters_CO2-2)

插件的版本。

![3](img/#co_helm_plugins_and_starters_CO2-3)

此插件的使用说明。

![4](img/#co_helm_plugins_and_starters_CO2-4)

插件的描述。

![5](img/#co_helm_plugins_and_starters_CO2-5)

特定平台的命令。如果客户端匹配特定的操作系统/架构组合，则运行该命令而不是默认命令。

![6](img/#co_helm_plugins_and_starters_CO2-6)

调用此插件时运行的命令。

![7](img/#co_helm_plugins_and_starters_CO2-7)

当作为插件的参数传递时，是否抑制传递给插件的 Helm 全局标志（例如`--debug`）。

![8](img/#co_helm_plugins_and_starters_CO2-8)

插件钩子（参见“Hooks”）。

![9](img/#co_helm_plugins_and_starters_CO2-9)

下载器及其相关协议（参见“Downloader Plugins”）。

插件的`name`将作为从 Helm CLI 调用此插件的子命令（例如`helm myplugin`）。因此，插件名称不应与任何现有的 Helm 子命令（如`install`、`repo`等）匹配。名称只能包含字符 a–z、A–Z、0–9、_ 和-。

插件的`version`应该是一个有效的 SemVer 2 版本。

插件的`usage`和`description`将在运行`helm help`和`helm help myplugin`时显示。但是，插件本身必须处理其自身的标志解析，例如`helm myplugin --help`。

当调用此插件时，`command`是 Helm 在子进程中执行的命令。如果定义了`platformCommands`部分，Helm 首先会检查系统是否与提供的`os`（操作系统）和`arch`（架构）匹配，如果匹配，则 Helm 将使用匹配条目中定义的`command`。`arch`字段是可选的，如果缺失，则仅检查`os`。

这是 Helm 确定在基于*plugin.yaml*内容和运行时环境时运行插件时要运行的命令的确切顺序：

1.  如果存在`platformCommand`，将首先搜索它。

1.  如果`os`和`arch`都与当前平台匹配，则搜索将停止并执行特定于平台的命令。

1.  如果`os`匹配并且没有更具体的匹配项，则将执行特定于平台的命令。

1.  如果没有找到`os`/`arch`匹配项，则将执行顶层默认的`command`。

1.  如果顶层没有`command`且`platformCommand`中没有匹配项，则 Helm 将以错误退出。

## 钩子

插件钩子允许在安装、更新或删除插件时采取额外的操作。

例如，您的插件的底层实现可能是一个平台特定的二进制文件，必须从互联网下载。该二进制文件的 URL 根据用户的操作系统而变化。

根据操作系统处理此逻辑的脚本可能如下所示：

```
#!/usr/bin/env bash

set -e

URL=""
EXTRACT_TO=""

if [[ "$(uname)" = "Darwin" ]]; then
    URL="https://example.com/releases/myplugin-mac"
    EXTRACT_TO="myplugin"
elif [[ "$(uname)" = "Linux" ]]; then
    URL="https://example.com/releases/myplugin-linux"
    EXTRACT_TO="myplugin"
else
    URL="https://example.com/releases/myplugin-windows"
    EXTRACT_TO="myplugin.exe"
fi

mkdir -p bin/
curl -sSL "${URL}" -o "bin/${EXTRACT_TO}"
```

通过为我们的插件定义安装钩子，我们可以使得此脚本在用户安装此插件时运行。

要定义一个钩子，请向您的*plugin.yaml*添加一个`hooks`部分，为您希望响应的每个事件定义命令：

```
...
hooks:
  install: "scripts/install-hook.sh" ![1](img/1.png)
  update: "scripts/update-hook.sh" ![2](img/2.png)
  delete: "scripts/delete-hook.sh" ![3](img/3.png)
```

![1](img/#co_helm_plugins_and_starters_CO3-1)

在运行`helm plugin install`命令时执行。

![2](img/#co_helm_plugins_and_starters_CO3-2)

在运行`helm plugin update`命令时执行。

![3](img/#co_helm_plugins_and_starters_CO3-3)

在运行`helm plugin remove`命令时执行。

## 下载器插件

一些插件具有特殊功能，允许它们作为下载图表的替代方案使用。

如果您以与纯图表存储方式不同的方式存储图表，或者如果您的图表存储库实现具有额外的要求，则此方法很有用。

*下载器插件*定义了一个（或多个）协议，如果在命令行中检测到，则指示 Helm 使用插件而不是 Helm 的内部下载机制下载*index.yaml*或 chart *.tgz*包。

下面是一个名为“super-secure”的下载器插件的*plugin.yaml*示例，它注册了`ss://`协议：

```
name: super-secure
version: 0.1.0
description: a super secure chart downloader
command: "${HELM_PLUGIN_DIR}/super-secure.sh"
downloaders:
  - command: "super-secure-downloader.sh" ![1](img/1.png)
    protocols: ![2](img/2.png)
      - "ss"
```

![1](img/#co_helm_plugins_and_starters_CO4-1)

插件作为下载器时调用的命令

![2](img/#co_helm_plugins_and_starters_CO4-2)

此插件声明的自定义协议

###### 注意

请记住，包括下载器插件在内的所有插件都定义了一个自定义的顶级命令（即 `helm super-secure`）。插件下载器的命令可以与 `command` 字段相同；但请注意，如果希望将插件同时用作标准插件和下载器，确定其用途可能会有一些挑战。您可以通过检查命令是否以四个命令行参数被调用来确定插件是否被用作下载器的一种方法。

下载器命令总是使用以下参数调用：

```
<command> certFile keyFile caFile full-URL
```

`certFile`、`keyFile` 和 `caFile` 参数来自一个 YAML 配置文件中的条目，其路径由 `$(helm env HELM_REPOSITORY_CONFIG)` 返回，并且在使用 `helm repo add` 添加仓库时设置（详见 第七章 了解更多背景）。`full-URL` 参数是正在下载的资源的完整 URL，可以是一个 *index.yaml* 文件，或者是一个图表 *.tgz*/*.prov* 文件。

让我们来看看由 `super-secure` 插件定义的 `ss://` 协议下载器的实现：

```
#!/usr/bin/env bash
set -e

# The fourth argument is the URL to the resource to download from the repo
URL="${4}"

# Replace "ss://" with "https://"
URL="$(echo ${URL} | sed 's/ss:/https:/')"

# Request the resource using the token, outputting contents to stdout
echo "Downloading $(basename ${URL}) using super-secure plugin..." 1>&2
curl -sL -H "Authorization: Bearer ${SUPER_SECURE_TOKEN}" "${URL}"
```

此下载器允许我们使用使用令牌/承载者认证保护的图表仓库。它期望环境变量 `SUPER_SECURE_TOKEN` 被设置，该令牌将用于构造请求图表仓库资源时使用的 `Authorization: Bearer <token>` 头部。

###### 注意

虽然 `super-secure` 插件是一个简单的下载器插件的很好示例，但是 Helm 的未来版本可能会直接支持 bearer token auth。

下载器插件期望将资源的内容输出到 `stdout`，因此任何额外的日志等应该打印到 `stderr`。这就是为什么在以 `echo` 开头的行中，我们使用 `1>&2` 将此消息重定向到 `stderr`。

安装此插件后，这是我们如何添加一个使用令牌认证保护的图表仓库的方法：

```
$ export SUPER_SECURE_TOKEN="abc123"
$ helm repo add my-secure-repo ss://secure.example.com
Downloading index.yaml using super-secure plugin...
"my-secure-repo" has been added to your repositories
```

此仓库 URL 现在将显示在本地仓库列表中，包含 `ss://` 协议：

```
$ helm repo list
NAME          	URL
my-secure-repo	ss://secure.example.com
```

现在该仓库可以像任何其他仓库一样使用，用于下载远程图表包：

```
$ export SUPER_SECURE_TOKEN="abc123"
$ helm pull my-secure-repo/superapp
Downloading superapp-0.1.0.tgz using super-secure plugin...
$ ls
superapp-0.1.0.tgz
```

下载器插件为 Helm 用户提供了一种通过定义自定义协议来扩展与图表仓库工作的传输机制的方式。当 Helm 检测到使用自定义协议时，它将尝试定位一个安装的插件来处理它，然后将资源请求推迟给该插件。

## 执行环境

由于插件旨在扩展 Helm 的功能，它们可能需要访问一些 Helm 的内部配置文件，或者在命令行上提供的全局标志。

为了让插件在运行时能够访问到这类信息，一系列已知的环境变量会被提供给插件。

这是当前所有插件可用的环境变量列表，按字母顺序排列：

`HELM_BIN`

正在执行的 Helm 命令的路径

`HELM_DEBUG`

全局布尔 `--debug` 选项设置的值（“true” 或 “false”）

`HELM_KUBECONTEXT`

全局 `--kube-context <context>` 选项设置的值

`HELM_NAMESPACE`

全局 `--namespace <namespace>` 选项设置的值

`HELM_PLUGIN_DIR`

当前插件的根目录

`HELM_PLUGIN_NAME`

当前插件的名称

`HELM_PLUGINS`

包含所有插件的顶层目录

`HELM_REGISTRY_CONFIG`

仓库配置的根目录

`HELM_REPOSITORY_CACHE`

仓库缓存的根目录

`HELM_REPOSITORY_CONFIG`

仓库配置的根目录

## Shell 自动完成

Helm 对于 Bash 和 Z shell（Zsh）都有内置的 shell 自动完成支持（参见 `helm completion --help`）。这在你无法记住正在尝试使用的子命令或标志名称时非常有帮助。

插件还可以通过两种方法（静态自动完成和动态完成）提供自定义的 shell 自动完成。

### 静态自动完成

通过在插件目录的根目录中包含名为 *completion.yaml* 的文件，Helm 插件可以静态指定插件可用的所有预期标志和命令。

这是一个虚构的 `zoo` 插件的 *completion.yaml* 示例：

```
name: zoo ![1](img/1.png)
flags: ![2](img/2.png)
  - disable-smells
  - disable-snacks
commands: ![3](img/3.png)
  - name: price ![4](img/4.png)
    flags:
      - kids-discount
  - name: animals
    commands:
      - name: list
        validArgs: ![5](img/5.png)
          - birds
          - reptiles
          - cats
      - name: describe
        flags:
          - format-json
        validArgs:
          - birds
          - reptiles
          - cats
```

![1](img/#co_helm_plugins_and_starters_CO5-1)

此完成文件所属插件的名称

![2](img/#co_helm_plugins_and_starters_CO5-2)

可用标志的列表（注意：这些标志不应包含 `-` 或 `--` 前缀）

![3](img/#co_helm_plugins_and_starters_CO5-3)

可用子命令的列表

![4](img/#co_helm_plugins_and_starters_CO5-4)

单个子命令的名称

![5](img/#co_helm_plugins_and_starters_CO5-5)

第一个子命令后面参数的有效选项列表

在顶层 `commands` 部分下面，可以为嵌套子命令指定另一个 `commands` 部分（以及递归指定多次）。每个 `commands` 部分中的命令都可以包含自己的 `flags` 和 `validArgs` 列表。

Helm 的全局标志，如 `--debug` 或 `--namespace`，已经由 Helm 内置的 shell 自动完成处理，因此不需要在 `flags` 下列出这些。

如果我们开始尝试运行示例 `zoo` 插件，然后按 Tab 键，它应该显示所有可用的子命令：

```
$ helm zoo  # (click tab)
animals  price
```

现在，如果我们做同样的操作，但在按下 Tab 键之前加上 `--disable-s` 后缀，我们应该能看到我们的标志：

```
$ helm zoo --disables-s  # (click tab)
--disable-smells  --disable-snacks
```

使用静态完成，我们能够与 Helm 现有的 shell 完成达到一致，使插件在 Helm 用户体验中感觉更加紧密集成。

###### 提示

如果您正在开发一个插件，必须打开一个新的终端窗口以刷新静态 shell 自动完成。

或者，您可以运行以下命令之一，在当前终端获取最新的完成：

```
source <(helm completion bash) # for Bash
source <(helm completion zsh)  # for Z shell
```

### 动态完成

在某些情况下，给定命令的有效参数可能事先不知道。例如，您可能希望为集群中的 Helm 发布名称提供作为插件有效参数的列表。这可以通过动态完成来实现。

要启用动态完成，请在插件目录的根目录中包含一个名为 *plugin.complete* 的可执行文件。此文件可以是任何类型的可执行文件，例如 Shell 脚本或二进制文件。

对于包含 *plugin.complete* 文件的插件，在请求完成时（例如按下 Tab 键），Helm 将运行此可执行文件，并将需要完成的文本作为第一个参数传递。该程序应返回一系列可能的结果，每个结果由新行分隔，并成功退出（即返回代码 0）。

您甚至可以决定将此完成功能作为主要插件程序的一部分提供，使用简单的包装脚本通过诸如 `--complete` 标志触发它。以下是执行此操作的基本 *plugin.complete* 可执行文件示例：

```
#!/usr/bin/env sh
$HELM_PLUGIN_DIR/my-plugin-program --complete "$@"
```

延续 `zoo` 插件示例，假设可用动物类别的列表不断变化，并存储在用户主目录中的名为 *animals.txt* 的文件中。以下是 *animals.txt* 可能的样子：

```
birds
reptiles
cats
```

我们希望能够根据此文件的内容动态提供完成功能。以下是一个示例，展示了一个可用于提供动态完成的 *plugin.complete* 可执行文件（Bash 脚本）：

```
#!/usr/bin/env bash
set -e
INPUT="${@}"
if [[ "${INPUT}" == "animals list"* ]]; then
    INPUT="$(echo "${INPUT}" | sed -e 's/^animals list //')"
    for flag in $(cat "${HOME}/animals.txt"); do
        if [[ "${flag}" == "${INPUT}"* ]]; then
            echo "${flag}"
        fi
    done
fi
```

现在，如果我们运行插件并输入 `animals list`，然后按 Tab 键，它应该显示所有可用的动物类别列表：

```
$ helm zoo animals list  # (press Tab key)
birds     cats      reptiles
```

为了确保它是动态的，让我们在 *animals.txt* 中添加一个额外的类别“monkeys”，然后再试一次：

```
$ echo "monkeys" >> "${HOME}/animals.txt"
$ helm zoo animals list  # (press Tab key)
birds     cats      monkeys   reptiles
```

它有效！

这只是使用动态完成的简单示例，但请记住，您还可以查询一些远程资源，例如 Kubernetes 集群中的资源，使其成为插件的强大功能。

###### 注意

如果已经使用 *completion.yaml* 文件进行静态完成，则不使用动态完成，即使插件的根目录中存在 `plugin.complete` 可执行文件。

# 起始程序

起始程序或起始包类似于 Helm 图表，不同之处在于它们被设计为新图表的模板。

当您使用 `helm create` 命令创建新图表时，它会生成一个使用 Helm 内置起始程序的新图表，这是一个使用最佳实践的通用图表。

要指定自定义起始程序，可以在创建新图表时使用 `--starter` 选项：

```
$ helm create --starter basic-webapp superapp
```

使用起始程序可以利用先前为类似目的的应用构建的图表。这对于即时准备部署到您的 Kubernetes 环境中的新项目非常有用。

## 将图表转换为起始程序

任何 Helm 图表都可以转换为启动器。唯一区别标准图表和启动器之间的唯一事物是启动器模板中对图表名称的动态引用的存在。

要将标准图表转换为启动器，请用字符串`<CHARTNAME>`替换对图表名称的任何硬编码引用。

为了演示，让我们从一个名为*mychart*的图表中获取这个简单的 ConfigMap 模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
data:
  hello: {{ .Values.hello | quote }}
```

这是模板在启动器中的样子：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
  labels:
    {{- include "<CHARTNAME>.labels" . | nindent 4 }}
data:
  hello: {{ .Values.hello | quote }}
```

###### 注意

此图表必须仍包含一个*Chart.yaml*文件才能工作；但是，它将被生成器覆盖。

## 使启动器可用于 Helm

在使用启动器之前，你必须首先为其决定一个唯一的名称，例如一个包含基本 Web 应用程序的启动器的“basic-webapp”。

要使此启动器在命令行上指定`--starter`标志时成为有效选项，它必须作为目录存在于文件路径`$(helm env HELM_DATA_HOME)/starters`中。

如果这是您添加的第一个启动器，请确保顶级*starters*目录首先存在：

```
$ export HELM_STARTERS="$(helm env HELM_DATA_HOME)/starters"
$ mkdir -p "${HELM_STARTERS}"
```

然后只需将整个*basic-webapp*目录复制到该顶级目录中：

```
cp -r basic-webapp "${HELM_STARTERS}"
```

## 使用启动器

一旦有了一个启动器，你可以通过在命令行上引用其名称来基于它生成新的图表：

```
$ helm create --starter basic-webapp superapp
Creating superapp
```

新生成的图表的结构将与启动器完全相同。启动器模板中所有对`<CHARTNAME>`的引用都将替换为新图表的名称（即*superapp*）。

这是基于一个启动器生成的图表的示例目录结构：

```
$ tree superapp/
superapp/
├── Chart.yaml
├── templates
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

从这里，你可以将这个新图表检入版本控制，并开始进行自定义以适应特定的应用程序。

# 进一步扩展 Helm

在本章中，我们讨论了如何使用插件和启动器扩展 Helm。然而，还有另一种方式可以扩展 Helm：通过开源贡献。

本书中的所有内容都是对 Helm 项目数千次开源贡献的反映。虽然大部分工作是由维护者（过去和现在）执行的，但大多数贡献来自全球各地的个人。这不仅包括对 Go 源代码的更改，还包括测试和文档更新。

您有想为 Helm 项目做出贡献的吗？请转到[Helm 社区登陆页面](https://oreil.ly/9TloH)了解更多信息！
