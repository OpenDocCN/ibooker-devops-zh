# 附录 D. 安装和配置 Git

*Git* 已成为跟踪文件更改的事实上的版本控制系统，并在处理 Kubernetes 内容时经常使用。与其他版本控制系统相比，由于其简化的分支管理功能而获得了流行。在开始在本地机器上使用 Git 之前，必须首先完成一系列步骤。

## D.1 安装 Git

要开始使用 Git 内容，必须安装 `git` 可执行文件。大多数操作系统都支持 `git`，包括 Linux、OSX 和 Windows，具体步骤可在 Git 网站上找到（[`git-scm.com/book/en/v2/Getting-Started-Installing-Git`](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)）。

当在 Linux 上安装 `git` 时，最简单的方法是使用包管理器，如 `apt` 或 `dnf`。在基于 Debian 的 Linux 操作系统上，请在终端中执行以下命令来安装 Git：

```
apt install git-all
```

在基于 RPM 的操作系统上，例如 Fedora 或 Red Hat Enterprise Linux，请在终端中执行以下命令来安装 Git：

```
dnf install git-all
```

通过检查版本来确认 Git 已成功安装：

```
git --version
```

如果返回了版本信息，则表示 Git 已成功安装。如果发生错误，请确认与安装方法相关的步骤已完全完成。

## D.2 配置 Git

尽管 Git 安装后即可使用，但建议采取一些额外的步骤来自定义 Git 环境。某些功能，如提交代码，在没有额外操作的情况下将不可用。

`git config` 子命令可用于检索和设置配置选项。这些选项可以在三个级别之一中指定：

+   在系统级别，并在 `[path]` `/etc/gitconfig` 文件中指定。

+   在用户配置文件级别，位于 `~/.gitconfig` 中。此级别可以通过使用 `git config` 子命令指定 `--global` 选项来定位。

+   在 `.git/config` 文件中的存储库级别。此级别可以通过使用 `git config` 子命令指定 `--local` 选项来定位。

虽然 Git 中提供了大量的可配置选项，但在安装 Git 时，有两个选项应该被定义：

+   用户名

+   电子邮件地址

这些值将与您执行的任何提交相关联。未能配置这些值将导致在尝试执行提交时出现以下错误。

列表 D.1 未配置 Git 身份时产生的错误信息

```
Author identity unknown

*** Please tell me who you are.

Run

  git config --global user.email "you@example.com"    ①
  git config --global user.name "Your Name"

to set your accounts default identity.
Omit --global to set the identity only in this repository.

fatal: unable to auto-detect email address (got 'root@machine.(none)')
```

① 表示应配置的变量

如 D.1 列表中描述的错误所示，`user.email` 和 `user.name` 变量都应该被配置。虽然这些变量可以在每个存储库中单独配置，但为了简单起见，在全局级别定义并适当修改各个存储库中的变量更为直接。

执行以下命令以设置 `user.email` 和 `user.name`，以配置所需的变量，并适当替换您的用户详细信息：

```
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

通过列出所有全局配置来确认值已适当地设置：

```
git config --global -l
```

应返回以下值：

```
user.name=Your Name
user.email=you@example.com
```

在此阶段，您的机器已准备好完全与 Git 生态系统进行交互。
