# 附录 C. 安装和配置 pip

与大多数其他编程语言一样，Python 包含了对启用可重用代码部分的支持，例如可以包含在其他应用程序中的语句和定义。这些可重用代码片段被称为 *模块*。多个模块可以组织在一起并包含在 *包* 中。标准 Python 库包含了一系列模块和包，这对于任何应用程序都是基本的。然而，标准 Python 库的内容并不涵盖所有可想象的使用案例。这就是用户定义的包和模块发挥作用的地方。随着越来越多的人创建定制的包，有一个简单的方法来分发和消费这些 Python 包变得非常重要。Python 包索引（PyPi）([`pypi.org/`](https://pypi.org/))试图提供一个解决方案，以提供一个集中位置来存储和发现 Python 社区共享的 Python 包。来自集中式 Python 包索引或自托管实例的 Python 包可以使用 *pip* 进行管理，pip 是一个包管理器，它简化了 Python 包的发现、下载和生命周期管理。

## C.1 安装 pip

存放在 Python 包索引中的 Python 包可以使用 `pip` 可执行文件进行管理。大多数最新的 Python 发行版（Python 2 的版本 >=2.9.2，Python 3 的版本 >=3.4）在二进制安装中预装了 pip。如果 Python 是通过其他方法安装的，例如包管理器，pip 可能没有被包含在内。您可以通过尝试执行 `pip`（如果您使用的是 Python 2 版本）或 `pip3`（如果您使用的是 Python 3 版本）来检查 pip 是否已安装。如果在执行前面的命令时返回错误，则必须安装 pip。

有几种方法可以安装 pip：

+   Python 3 的 `ensurepip` 模块

+   `get-pip.py` 脚本

+   包管理器

让我们使用 `ensurepip` Python 模块来安装 pip，因为它使用了适用于大多数平台的本地 Python 构造。执行以下命令来安装 pip：

```
python -m ensurepip --upgrade
```

通过执行 `pip` 命令来确认 pip 已安装：

```
pip
```

如果命令没有错误返回，则表示 pip 已成功安装。

注意：在某些系统上，可能使用别名或符号链接将 `python3` 和 `python` 可执行文件链接起来，以提供向后兼容性或简化与 Python 的交互；同样适用于 `pip`。在执行 `python`、`python3`、`pip` 或 `pip3` 时指定 `--version` 标志可以确认正在使用的特定版本。

## C.2 基本 pip 操作

在开始使用 pip 之前，建议您更新它及其支持的工具到最新版本。获取最新更新将确保能够适当访问任何所需的源存档。执行以下命令以更新 pip，以及 `setuptools` 和 `wheel` 包到最新版本：

```
python -m pip install --upgrade pip setuptools wheel
```

在更新了必要的工具后，让我们开始使用 pip。

对于任何使用包管理器的人来说，第一步是确定要安装的软件组件。这可能事先已知，或者可能需要从可用组件列表中查询。搜索包的最佳位置是 PyPi 网站([`pypi.org/`](https://pypi.org/))，该网站详细介绍了每个可用的包、它们的历史和任何依赖项。

从 Python 索引中可安装的包数不胜数，选择正确的包可能是一项挑战。Python 开发者最常见的用例之一是进行基于 HTTP 的请求。虽然 Python 提供了如`http.client`等模块，但构建简单的查询可能相当复杂。`requests`模块试图简化基于 HTTP 的调用。

与 Requests 模块相关的详细信息可以在 PyPi 网站上找到，但让我们使用 pip 来安装包。执行以下命令使用 pip 安装`requests`包：

```
pip install requests
```

使用`info`子命令可以查看已安装包的相关信息：

```
pip show requests
```

命令的响应如下：

```
Name: requests
Version: 2.26.0
Summary: Python HTTP for Humans.
Home-page: https://requests.readthedocs.io
Author: Kenneth Reitz
Author-email: me@kennethreitz.org
License: Apache 2.0
Location: /usr/local/lib/python3.9/site-packages
Requires: certifi, charset-normalizer, idna, urllib3
Required-by:
```

安装了`requests`包后，可以使用简单的 Python 交互会话来说明其用法。执行以下命令以首先启动 Python 交互会话：

```
python
```

现在导入 requests，查询远程地址并打印 HTTP 状态。

列表 C.1 使用 requests 包查询远程服务器

```
import requests                                 ①
response = requests.get("https://google.com")   ②
print(response.status_code)                     ③
```

① 导入 requests 包。

② 执行请求。

③ 打印请求的 HTTP 状态。

应返回`200`响应代码，表示成功查询远程 HTTP 服务器。输入`exit()`退出 Python 交互控制台。

已安装的 Python 包也可以通过 pip 安装。可以使用`pip list`命令确定当前已安装的哪些包。一旦确定了要删除的所需包，就可以使用`pip uninstall`命令来删除。要删除之前安装的 requests 包，请执行以下命令。

列表 C.2 删除 requests 包

```
pip uninstall -y requests       ①
```

① 使用-y 标志将跳过在删除包之前的确认提示。

一旦命令成功完成，Python 包已被删除。
