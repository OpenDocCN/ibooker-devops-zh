# 第六章：字段和过滤器

终于是时候将你在前几章学到的所有理论付诸实践了。在本章中，你将学习 Falco 过滤器的相关知识：它们是什么，如何工作以及如何使用它们。

过滤器是 Falco 的核心。它们也是一个强大的调查工具，可以在多种其他工具中使用，比如 sysdig。因此，我们预期你即使在完成本书后，也会经常回来查阅本章内容——因此我们设计它可以用作参考。例如，它包含了过滤语言提供的所有运算符和数据类型的表格，设计用于快速查询，以及 Falco 最有用字段的详细文档列表。本章的内容几乎每次你编写 Falco 规则时都会派上用场，因此确保将其加入书签！

# 什么是过滤器？

让我们从一个半正式的定义开始：

> Falco 中的*过滤器*是一个包含一系列比较的条件，这些比较由布尔运算符连接。每个比较评估从输入事件中提取的字段与一个常量之间的关系运算符。过滤器中的比较按从左到右的顺序进行评估，但可以使用括号定义优先级。过滤器应用于输入事件，并返回一个布尔结果，指示事件是否与过滤器匹配。

哎呀。这个描述非常枯燥并且有些复杂。但如果我们借助一些例子来解释，你会发现并不难。让我们从第一句开始：

> Falco 中的*过滤器*是一个包含一系列比较的条件，这些比较由布尔运算符连接。

这只是意味着一个过滤器看起来像这样：

```
A = B and not C != D
```

换句话说，如果你可以在任何编程语言中编写一个`if`条件，过滤器语法看起来会非常熟悉。接下来的句子是：

> 每个比较评估从输入事件中提取的字段与一个常量之间的关系运算符。

这告诉我们，Falco 的过滤语法基于*字段*的概念，我们将在本章后面详细描述。字段名称采用点语法，并出现在每个比较的左侧。右侧是将与字段进行比较的常量值。以下是一个示例：

```
proc.name = emacs or proc.pid != 1234
```

继续下一个：

> 过滤器中的比较按从左到右的顺序进行评估，但可以使用括号定义优先级。

这意味着你可以使用括号来组织你的过滤器。例如：

```
proc.name = emacs or (proc.name = vi and container.name=redis)
```

同样，这与在你喜爱的编程语言中使用逻辑表达式内部的括号完全相同。现在是最后一句：

> 过滤器应用于输入事件，并返回一个布尔结果，指示事件是否与过滤器匹配。

当您在 Falco 规则中指定一个过滤器时，该过滤器将应用于每个输入事件。例如，如果您使用 Falco 的驱动程序之一，则过滤器将应用于每个系统调用。过滤器评估系统调用并返回布尔值：`true`表示事件满足过滤器（我们说过滤器*匹配*事件），而`false`表示过滤器拒绝或丢弃事件。例如，此过滤器：

```
proc.name = emacs or proc.name = vi
```

匹配（返回`true`），由名为`emacs`或`vi`的进程生成的每个系统调用。

这基本上就是您需要在高层次上了解的全部。现在让我们深入了解细节。

# 过滤语法参考

从语法角度来看，正如我们提到的，编写 Falco 过滤器非常类似于在任何编程语言中编写`if`条件，因此如果您具有基本的编程经验，不应该期望有任何重大的惊喜。然而，有一些区域是特定于您在 Falco 中进行匹配类型的。本节详细讨论了语法，为您提供了全面的图片。

## 关系运算符

表 6-1 提供了所有可用关系运算符的参考，包括每个运算符的示例。

表 6-1\. Falco 的关系运算符

| 运算符 | 描述 | 示例 |
| --- | --- | --- |
| `=`, `!=` | 一般的相等性/不等性运算符。可用于所有类型的字段。 | `proc.name = emacs` |
| `<=`, `<`, `>=`, `>` | 数字比较运算符。仅可用于数字字段。 | `evt.buflen > 100` |
| `contains` | 仅可用于字符串字段。对字段值进行区分大小写的字符串搜索，如果字段值包含指定常量则返回`true`。 | `fd.filename 包含 passwd` |
| `icontains` | 类似于`contains`，但不区分大小写。 | `user.name 包含 john` |
| `bcontains` | 类似于`contains`，但允许您在二进制缓冲区上执行检查。 | `evt.buf bcontains DEADBEEF` |
| `startswith` | 仅可用于字符串字段。如果指定的常量与字段值的开头匹配则返回`true`。 | `fd.directory 开始于 "/etc"` |
| `bstartswith` | 类似于`startswith`，但允许您在二进制缓冲区上执行检查。 | `evt.buf bstartswith DEADBEEF` |
| `endswith` | 仅可用于字符串字段。如果指定的常量与字段值的结尾匹配则返回`true`。 | `fd.filename 结尾为 ".key"` |
| `in` | 将字段值与多个常量进行比较，如果其中一个或多个常量等于字段值则返回`true`。可用于所有字段，包括数字字段和字符串字段。 | `proc.name 在 (vi, emacs)` |
| `intersects` | 当具有多个值的字段包含至少一个与提供的常量之一匹配的值时返回`true`。 | `ka.req.pod.volumes.hostpath 交集 (/proc, /var/run/docker.sock)` |

| `pmatch` | 如果常量之一是字段值的前缀，则返回 `true`。注意：`pmatch` 可用作 `in` 运算符的替代方案，并且在大量常数的情况下执行效果更好，因为它在内部实现为前缀树，而不是多个比较。 | `fd.name pmatch (/var/run, /etc, /lib, /usr/lib)` `fd.name = /var/run/docker` 成功，因为 `/var/run` 是 `/var/run/docker` 的前缀。

`fd.name = /boot` 不能成功，因为没有任何常数是 `/boot` 的前缀。

`fd.name = /var` 不能成功，因为没有任何常数是 `/var` 的前缀。 |

| `exists` | 如果输入事件存在给定字段，则返回 `true`。 | `evt.res exists` |
| --- | --- | --- |
| `glob` | 根据 Unix shell 通配符模式将给定字符串与字段值匹配。有关详细信息，请在终端中输入 `**man 7 glob**`。 | `fd.name glob '/home/*/.ssh/*'` |

## 逻辑运算符

您可以在 Falco 过滤器中使用的逻辑运算符非常直接，不包含任何意外。表 6-2 列出了它们并提供了示例。

表格 6-2\. Falco 的逻辑运算符

| 运算符 | 示例 |
| --- | --- |
| `and` | `proc.name = emacs and proc.cmdline contains myfile.txt` |
| `or` | `proc.name = emacs or proc.name = vi` |
| `not` | `not proc.name = emacs` |

## 字符串和引用

字符串常量可以不使用引号指定：

```
proc.name = emacs
```

引号可以用于括起包含空格或特殊字符的字符串。单引号和双引号都被接受。例如：

```
proc.name = "my process" or proc.name = 'my process'
```

这意味着您可以在字符串中包含引号：

```
evt.buffer contains '"'
```

# 字段

如您所见，Falco 过滤器并不复杂。但是，它们非常灵活和强大。这种强大来自于您可以在过滤条件中使用的字段。Falco 为您提供访问多种字段的权限，每个字段公开了 Falco 捕获的输入事件的属性。由于字段非常重要，让我们看看它们是如何工作和组织的。然后我们将讨论何时以及使用哪些字段。

## 参数字段与丰富字段

字段将输入事件的属性公开为类型化值。例如，字段可以是字符串（如进程名称）或数字（如进程 ID）。

在最高级别上，Falco 提供了两类字段。第一类包括通过解析输入事件获得的字段。系统调用参数，例如 `open` 系统调用的文件名或 `read` 系统调用的缓冲区参数，都是此类字段的示例。您可以使用以下语法访问这些字段，其中 `*X*` 是要访问的参数的名称：

```
evt.arg.*X*
```

或者，其中 `*N*` 是参数的位置：

```
evt.arg[*N*]
```

例如：

```
evt.arg.name = /etc/passwd
evt.arg[1] = /etc/passwd
```

要了解特定事件类型支持哪些参数，请使用 sysdig。sysdig 中事件的输出行将显示所有参数及其名称。

第二类别包括从*libsinsp*捕获系统调用和其他事件时执行的丰富化过程中派生的字段，详见第五章。Falco 导出许多字段，这些字段公开了*libsinsp*的线程和文件描述符表的内容，为从驱动程序接收的事件添加了丰富的上下文。

为了帮助您理解它是如何工作的，让我们以`proc.cwd`字段为例。对于 Falco 捕获的每个系统调用，此字段包含发出系统调用的进程的当前工作目录。如果您想捕获当前在特定目录内运行的所有进程生成的系统调用，这非常方便；例如：

```
proc.cwd = /tmp
```

进程的工作目录不是系统调用的一部分，因此要公开此字段，需要跟踪进程的工作目录，并将其附加到进程生成的每个系统调用中。这反过来涉及四个步骤：

1.  当一个进程启动时，收集其工作目录，并将其存储在线程表中的进程条目中。

1.  跟踪进程何时更改其工作目录（通过拦截和解析`chdir`系统调用），并相应地更新线程表条目。

1.  解析每个系统调用的线程 ID，以识别相应的线程表条目。

1.  返回线程表条目的`cwd`值。

*libsinsp*做了所有这些工作，这意味着`proc.cwd`字段可用于每个系统调用，而不仅仅是像`chdir`这样与目录相关的调用。Falco 为向您公开此字段所做的大量工作令人印象深刻！

基于丰富化的过滤非常强大，因为它允许您根据并非包含在系统调用本身中但对安全策略非常有用的属性来过滤系统调用（以及任何其他事件）。例如，以下过滤器允许您捕获读取或写入*/etc/passwd*的系统调用：

```
evt.is_io=true and fd.name=/etc/passwd
```

即使这些系统调用最初不包含任何有关文件名的信息（它们操作文件描述符），它们也能正常工作。箱中提供的数百种基于丰富化的字段是 Falco 如此强大和多功能的主要原因。

## 强制字段与可选字段

一些字段存在于每个输入事件中，无论事件类型或族群如何，您都能保证找到它们。此类字段的示例包括`evt.ts`、`evt.dir`和`evt.type`。

然而，大多数字段是可选的，并且仅存在于某些输入事件类型中。通常情况下，你不需要担心这一点，因为不存在的字段会在不生成错误的情况下仅评估为`false`。例如，以下检查将对所有没有名为`name`的参数的事件评估为`false`：

```
evt.arg.name contains /etc
```

但在某些情况下，您可能想要显式检查字段是否存在。一个原因是解决像`evt.arg.name != /etc`这样的模糊性，以确定对于没有名为`name`的参数的事件，是否返回`true`或`false`。您可以通过使用`exists`关系运算符来回答这类问题：

```
evt.arg.name exists and evt.arg.name != /etc
```

## 字段类型

字段具有类型，用于验证值并确保过滤器的语法正确性。看下面的过滤器：

```
proc.pid = hello
```

Falco 和 sysdig 将使用以下错误拒绝这个：

```
filter error at position 16: hello is not a valid number
```

之所以会发生这种情况是因为`proc.pid`字段的类型为`INT64`，所以其值必须是整数。类型系统还允许 Falco 通过理解字段背后的含义来改善某些字段的渲染。例如，`evt.arg.res`的类型是`ERRNO`，默认情况下是一个数字。然而，可能时，Falco 会将其解析为一个错误代码字符串（如`EAGAIN`），从而提高字段的可读性和可用性。

当我们研究关系运算符时，我们注意到一些与大多数编程语言中的运算符非常相似，而其他一些则是 Falco 过滤器独有的。字段类型也是如此。表 6-3 列出了您在 Falco 过滤器字段中可能遇到的类型。

表 6-3\. 字段类型

| 类型 | 描述 |
| --- | --- |
| `INT8`, `INT16`, `INT32`, `INT64`, `UINT8`, `UINT16`, `UINT32`, `UINT64`, `DOUBLE` | 像您喜欢的编程语言中的数值类型。 |
| `CHARBUF` | 可打印字符缓冲区。 |
| `BYTEBUF` | 一个原始字节缓冲区，不适合打印。 |
| `ERRNO` | 一个`INT64`值，可能时，会被解析为错误代码。 |
| `FD` | 一个`INT64`值，可能时，会被解析为文件描述符的值。例如，对于文件，这会被解析为文件名；对于套接字，这会被解析为 TCP 连接元组。 |
| `PID` | 一个`INT64`值，可能时，会被解析为进程名称。 |
| `FSPATH` | 包含相对或绝对文件系统路径的字符串。 |
| `SYSCALLID` | 一个 16 位系统调用 ID。可能时，该值会被解析为系统调用名称。 |
| `SIGTYPE` | 一个 8 位信号编号，可能时，会被解析为信号名称（例如`SIGCHLD`）。 |
| `RELTIME` | 一个相对时间，精确到纳秒级，呈现为可读字符串。 |
| `ABSTIME` | 绝对时间间隔。 |
| `PORT` | 一个 TCP/UDP 端口。可能时，会被解析为协议名称。 |
| `L4PROTO` | 一个 1 字节的 IP 协议类型。可能时，会解析为 L4 协议名称（TCP, UDP）。 |
| `BOOL` | 一个布尔值。 |
| `IPV4ADDR` | 一个 IPv4 地址。 |
| `DYNAMIC` | 表示字段类型根据上下文可变。用于像`evt.rawarg`这样的通用字段。 |
| `FLAGS8`, `FLAGS16`, `FLAGS32` | 标志字（即，使用二进制编码的一组标志作为数字）。在可能的情况下，将其转换为可读字符串（例如，`O_RDONLY&#124;O_CLOEXEC`）。字符串的解析取决于上下文，因为事件可以注册自己的标志值。因此，例如，lseek 系统调用事件的标志将转换为`SEEK_END`，`SEEK_CUR`等值，而`sockopt`的标志将转换为`SOL_SOCKET`，`SOL_TCP`等等。 |
| `UID` | 当可能时，将 Unix 用户 ID 解析为用户名。 |
| `GID` | 当可能时，将 Unix 组 ID 解析为组名。 |
| `IPADDR` | 一个 IPv4 或 IPv6 地址。 |
| `IPNET` | 一个 IPv4 或 IPv6 网络。 |
| `MODE` | 用于表示文件模式的 32 位位掩码。 |

您如何找出要使用的字段的类型？最好的方法是使用 Falco 的`--list`和`-v`选项调用：

```
$ falco --list -v
```

这将打印字段的完整列表，包括每个条目的类型信息。

# 使用字段和过滤器

现在您已经了解了过滤器和字段，让我们看看如何在实践中使用它们。我们将重点放在 Falco 和 sysdig 上。

## Falco 中的字段和过滤器

字段和过滤器是 Falco 规则的核心。字段用于表达规则的条件，既是条件的一部分，也是输出的一部分。为了演示如何使用它们，我们将制定我们自己的规则。

假设我们希望 Falco 在每次尝试更改文件权限并使其对其他用户可执行时通知我们。发生这种情况时，我们想知道已更改的文件的名称，文件的新模式以及导致问题的用户的名称。我们还想知道模式更改尝试是否成功。

这是规则：

```
- rule: File Becoming Executable by Others
  desc: Attempt to make a file executable by other users
  condition: >
    (evt.type=chmod or evt.type=fchmod or evt.type=fchmodat) 
    and evt.arg.mode contains S_IXOTH
  output: >
    attempt to make a file executable by others 
    (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name 
    failed=%evt.failed)
  priority: WARNING
```

`condition`部分是指定规则过滤器的地方。

文件模式，包括可执行位，是使用`chmod`系统调用或其变体进行更改的。因此，过滤器的第一部分选择了类型为`chmod`，`fchmod`或`fchmodat`的事件：

```
evt.type=chmod or evt.type=fchmod or evt.type=fchmodat
```

现在我们已经有了正确的系统调用，我们想要接受仅设置了“其他”可执行位的子集。阅读[`chmod`手册页](https://oreil.ly/zuKuC)显示我们需要检查的标志是`S_IXOTH`。我们通过使用`contains`操作符来确定其是否存在：

```
evt.arg.mode contains S_IXOTH
```

将这两个片段组合并使用`and`得到完整的过滤器。简单！

现在，让我们将注意力集中在规则的`output`部分。这是我们告诉 Falco 当规则条件返回`true`时在屏幕上打印什么的地方。您会注意到，这只是一个类似于`printf`的字符串，其中混合了常规文本和字段，这些字段的值将在最终消息中解析：

```
attempt to make a file executable by others (file=%evt.arg.filename 
mode=%evt.arg.mode user=%user.name failed=%evt.failed)
```

您唯一需要记住的是，在输出字符串中需要使用`%`字符作为字段名的前缀；否则，它们将仅被视为字符串的一部分。

是时候让您尝试一下了！将前述规则保存在名为*ch6.yaml*的文件中。之后，在终端中运行以下命令行：

```
$ sudo falco -r ch6.yaml
```

然后，在另一个终端中，运行以下两个命令：

```
$ echo test > test.txt
$ chmod o+x test.txt
```

这是您将在 Falco 终端中获得的输出：

```
17:26:43.796934201: Warning attempt to make a file executable by others 
(file=/home/loris/test.txt mode=S_IXOTH|S_IWOTH|S_IROTH|S_IXGRP|S_IWGRP
|S_IRGRP|S_IXUSR|S_IWUSR|S_IRUSR user=root failed=false)
```

恭喜，您刚刚执行了自己的 Falco 检测！请注意`evt.arg.mode`和`evt.failed`如何以人类可读的方式显示，即使在内部它们是数字。这显示了过滤器/字段类型系统的强大功能。

## sysdig 中的字段和过滤器

在第四章中提供了 sysdig 的简介（如果您需要复习，请参见“sysdig”）。这里我们将特别看看 sysdig 中如何使用过滤器和字段。

虽然 Falco 基于规则的概念并在规则匹配时通知用户，sysdig 专注于调查、故障排除和威胁狩猎工作流程。在 sysdig 中，您可以使用过滤器来*限制*输入，并且（可选地）使用字段格式化来*控制*输出。这两者的结合为调查提供了极大的灵活性。

在 sysdig 中，过滤器是在命令行末尾指定的：

```
$ sudo sysdig proc.name=echo
```

使用`-p`命令行标志提供输出格式化，并使用与我们刚刚在讨论 Falco 输出时描述的相同的`printf`-类似语法：

```
$ sudo sysdig -p"type:%evt.type proc:%proc.name" proc.name=echo
```

请记住的一件重要事情是，当使用`-p`标志时，sysdig 只会为*所有*指定过滤器存在的事件打印输出行。因此，这个命令：

```
$ sudo sysdig -p"%evt.res %proc.name"
```

仅为具有返回值*和*进程名称的事件打印一行，例如跳过所有系统调用“enter”事件。如果您关心查看所有事件，请在格式字符串的开头放置星号（`*`）：

```
$ sudo sysdig -p"*%evt.res %proc.name"
```

当字段缺失时，它将显示为`<NA>`。

当使用`-p`未指定格式时，sysdig 以标准格式显示输入事件，方便地包括所有参数和参数名，用于每个系统调用。以下是一个`openat`系统调用的 sysdig 输出行示例，其中以粗体突出显示系统调用参数以提高可见性：

```
4831 20:50:01.473556825 2 cat (865.865) < openat fd=7(<f>/tmp/myfile.txt) 
dirfd=-100(AT_FDCWD) name=/tmp/myfile.txt flags=1(O_RDONLY) mode=0 dev=4
```

每个参数都可以使用`evt.arg`语法在过滤器中使用：

```
$ sudo sysdig evt.arg.name=/tmp/myfile.txt
```

作为更高级的示例，让我们将我们在前一节为 Falco 创建的*文件被其他人设置为可执行*规则转换为 sysdig 命令行：

```
$ sudo sysdig -p"attempt to make a file executable by others \
  (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name \
  failed=%evt.failed)" \ 
  "(evt.type=chmod or evt.type=fchmod or evt.type=fchmodat) \ 
  and evt.arg.mode contains S_IXOTH"
```

这展示了在创建新规则时如何将 sysdig 作为开发工具使用的简便性。

# Falco 的最有用字段

本节介绍了按类别组织的一些最重要的 Falco 字段的精选列表。您可以在编写过滤器时将此列表作为参考。要获取包括所有插件字段的完整列表，请在命令行中使用以下命令：

```
$ falco --list -v
```

## 一般情况

列在表 6-4 中的字段适用于每个事件，并包括事件的一般属性。

表 6-4\. `evt`过滤器类字段

| 字段名 | 描述 |
| --- | --- |
| `evt.num` | 事件编号。 |
| `evt.time` | 事件时间戳，包括纳秒部分的字符串。 |
| `evt.dir` | 事件方向；可以是 `>` 表示进入事件，或 `<` 表示退出事件。 |
| `evt.type` | 事件名称（例如 `open`）。 |
| `evt.cpu` | 发生此事件的 CPU 编号。 |
| `evt.args` | 所有事件参数，聚合为单个字符串。 |
| `evt.rawarg` | 事件参数之一，按名称指定（例如 `evt.rawarg.fd`）。 |
| `evt.arg` | 事件参数之一，按名称或编号指定。某些事件（如返回代码或文件描述符）将在可能时转换为文本表示（例如 `evt.arg.fd` 或 `evt.arg[0]`）。 |
| `evt.buffer` | 事件的二进制数据缓冲区（例如 read、recvfrom 等）。在过滤器中使用 `contains` 来搜索 I/O 数据缓冲区。 |
| `evt.buflen` | 具有二进制数据缓冲区的事件的缓冲区长度，如 `read`、`recvfrom` 等。 |
| `evt.res` | 事件返回值，作为字符串。如果事件失败，则结果是错误代码字符串（例如 `ENOENT`）；否则，结果是字符串 `SUCCESS`。 |
| `evt.rawres` | 事件返回值，作为数字（例如 `-2`）。用于范围比较时很有用。 |
| `evt.failed` | 对于返回错误状态的事件为 `true`。 |

## Processes

此类中的字段包含关于进程和线程的所有信息。 Table 6-5 中的信息主要来自内存中 *libsinsp* 构建的进程表。

Table 6-5\. `proc` 过滤器类字段

| 字段名 | 描述 |
| --- | --- |
| `proc.pid` | 生成事件的进程 ID。 |
| `proc.exe` | 第一个命令行参数（通常是可执行文件名或自定义名称）。 |
| `proc.name` | 生成事件的可执行文件的名称（不包括路径）。 |
| `proc.args` | 启动生成事件进程时传递的命令行参数。 |
| `proc.env` | 生成事件的进程的环境变量。 |
| `proc.cwd` | 事件的当前工作目录。 |
| `proc.ppid` | 生成事件的进程的父进程 PID。 |
| `proc.pname` | 生成事件进程的父进程的名称（不包括路径）。 |
| `proc.pcmdline` | 生成事件进程的父进程的完整命令行（`proc.name` + `proc.args`）。 |
| `proc.loginshellid` | 当前进程祖先中最老的 shell 的 PID（如果存在）。此字段可用于区分不同的用户会话，并与像 spy_user 这样的凿子一起使用。 |
| `thread.tid` | 生成事件的线程 ID。 |
| `thread.vtid` | 生成事件的线程 ID，在其当前 PID 命名空间中可见。 |
| `proc.vpid` | 生成事件的进程 ID，在其当前 PID 命名空间中可见。 |
| `proc.sid` | 生成事件的进程的会话 ID。 |
| `proc.sname` | 当前进程的会话领导者的名称。这要么是具有`pid=proc.sid`的进程，要么是具有与当前进程相同会话 ID 的最年长的祖先。 |
| `proc.tty` | 进程的控制终端。对于没有终端的进程，这是`0`。 |

## 文件描述符

表 6-6 列出了与文件描述符相关的字段，这些字段是 I/O 的基础。包含有关文件和目录、网络连接、管道和其他类型的进程间通信的详细信息的字段都可以在这个类中找到。

表 6-6\. `fd`过滤类字段

| 字段名 | 描述 |
| --- | --- |
| `fd.num` | 标识文件描述符的唯一编号。 |
| `fd.typechar` | 文件描述符的类型，以单个字符表示。可以是`f`表示文件，`4`表示 IPv4 套接字，`6`表示 IPv6 套接字，`u`表示 Unix 套接字，`p`表示管道，`e`表示 eventfd，`s`表示 signalfd，`l`表示 eventpoll，`i`表示 inotify，或`o`表示未知。 |
| `fd.name` | 文件描述符的完整名称。如果是文件，则此字段包含完整路径。如果是套接字，则此字段包含连接元组。 |
| `fd.directory` | 如果文件描述符是文件，则包含它的目录。 |
| `fd.filename` | 如果文件描述符是文件，则是不带路径的文件名。 |
| `fd.ip` | *(仅过滤)* 匹配文件描述符的 IP 地址（客户端或服务器）。 |
| `fd.cip` | 客户端的 IP 地址。 |
| `fd.sip` | 服务器的 IP 地址。 |
| `fd.lip` | 本地 IP 地址。 |
| `fd.rip` | 远程 IP 地址。 |
| `fd.port` | *(仅过滤)* 匹配文件描述符的端口（客户端或服务器）。 |
| `fd.cport` | 对于 TCP/UDP 文件描述符，客户端的端口。 |
| `fd.sport` | 对于 TCP/UDP 文件描述符，服务器的端口。 |
| `fd.lport` | 对于 TCP/UDP 文件描述符，本地端口。 |
| `fd.rport` | 对于 TCP/UDP 文件描述符，远程端口。 |
| `fd.l4proto` | 套接字的 IP 协议。可以是`tcp`，`udp`，`icmp`或`raw`。 |

## 用户和用户组

表 6-7 列出了`user`和`group`过滤类中的字段。

表 6-7\. `user`和`group`过滤类字段

| 字段名 | 描述 |
| --- | --- |
| `user.uid` | 用户的 ID |
| `user.name` | 用户的名称 |
| `group.gid` | 用户组的 ID |
| `group.name` | 用户组的名称 |

## 容器

`container`类中的字段（表 6-8）可用于与容器相关的一切，包括获取 ID、名称、标签和挂载。

表 6-8\. `container`过滤类字段

| 字段名 | 描述 |
| --- | --- |
| `container.id` | 容器 ID。 |
| `container.name` | 容器名称。 |
| `container.image` | 容器镜像名称（例如，Docker 中的`falcosecurity/falco:latest`）。 |
| `container.image​.id` | 容器镜像 ID（例如，`6f7e2741b66b`）。 |
| `container​.privi⁠leged` | 运行为特权的容器为 `true`，否则为 `false`。 |
| `con⁠tainer​.mounts` | 一组以空格分隔的挂载信息。列表中每个项的格式为 `*<source>*:*<dest>*:*<mode>*:*<rdrw>*:*<propagation>*`。 |
| `container.mount` | 单个挂载的信息，由编号（例如，`container.mount[0]`）或挂载源（例如，`con⁠tainer.mount[/usr/local]`）指定。路径名可以是通配符（例如，`container.mount[/usr/local/*]`），在这种情况下，将返回第一个匹配的挂载。信息的格式为 `*<source>*:*<dest>*:*<mode>*:*<rdrw>*:*<propagation>*`。如果没有指定索引或匹配提供的源的挂载，则返回字符串 `"none"` 而不是 NULL 值。 |
| `container.image​.reposi⁠tory` | 容器镜像仓库（例如，`falcosecurity/falco`）。 |
| `con⁠tainer.image​.tag` | 容器镜像标签（例如，`stable`，`latest`）。 |
| `con⁠tainer.image​.digest` | 容器镜像注册表摘要（例如，`sha256:d977378f890d445c15e51795296​e4e5062f109ce6da83e0a355fc4ad8699d27`）。 |

## Kubernetes

当 Falco 配置为与 Kubernetes API 服务器接口时，可以使用此类中的字段（列在 Table 6-9）来获取有关 Kubernetes 对象的信息。

Table 6-9\. `k8s` 过滤器类字段

| 字段名称 | 描述 |
| --- | --- |
| `k8s.pod.name` | Kubernetes Pod 名称。 |
| `k8s.pod.id` | Kubernetes Pod ID。 |
| `k8s.pod.label` | Kubernetes Pod 标签（例如，`k8s.pod.label.foo`）。 |
| `k8s.rc.name` | Kubernetes ReplicationController 名称。 |
| `k8s.rc.id` | Kubernetes ReplicationController ID。 |
| `k8s.rc.label` | Kubernetes ReplicationController 标签（例如，`k8s.rc.label.foo`）。 |
| `k8s.svc.name` | Kubernetes 服务名称。可能返回多个值，已连接。 |
| `k8s.svc.id` | Kubernetes Service ID。可能返回多个值，已连接。 |
| `k8s.svc.label` | Kubernetes Service 标签（例如，`k8s.svc.label.foo`）。可能返回多个值，已连接。 |
| `k8s.ns.name` | Kubernetes 命名空间名称。 |
| `k8s.ns.id` | Kubernetes 命名空间 ID。 |
| `k8s.ns.label` | Kubernetes 命名空间标签（例如，`k8s.ns.label.foo`）。 |
| `k8s.rs.name` | Kubernetes ReplicaSet 名称。 |
| `k8s.rs.id` | Kubernetes ReplicaSet ID。 |
| `k8s.rs.label` | Kubernetes ReplicaSet 标签（例如，`k8s.rs.label.foo`）。 |
| `k8s.deploy⁠ment​.name` | Kubernetes 部署名称。 |
| `k8s.deployment.id` | Kubernetes 部署 ID。 |
| `k8s.deployment.label` | Kubernetes 部署标签（例如，`k8s.rs.label.foo`）。 |

## CloudTrail

当配置 CloudTrail 插件时，可以使用 `cloudtrail` 类中的字段（列在 Table 6-10）来构建 AWS 检测的过滤器和格式化程序。

Table 6-10\. `cloudtrail` 过滤器类字段

| 字段名称 | 描述 |
| --- | --- |
| `ct.error` | 事件的错误代码。如果没有错误，则为 `""`。 |
| `ct.src` | CloudTrail 事件的来源（在 JSON 中为 `eventSource`）。 |
| `ct.shortsrc` | CloudTrail 事件的来源（在 JSON 中为 `eventSource`），不包括 `.amazonaws.com` 后缀。 |
| `ct.name` | CloudTrail 事件的名称（在 JSON 中为 `eventName`）。 |
| `ct.user` | CloudTrail 事件的用户（在 JSON 中为 `userIdentity.userName`）。 |
| `ct.region` | CloudTrail 事件的区域（在 JSON 中为 `awsRegion`）。 |
| `ct.srcip` | 生成事件的 IP 地址（在 JSON 中为 `sourceIPAddress`）。 |
| `ct.useragent` | 生成事件的用户代理（在 JSON 中为 `userAgent`）。 |
| `ct.readonly` | 如果事件仅读取信息（例如 `DescribeInstances`），则为 `true`；如果事件修改状态（例如 `RunInstances`、`CreateLoadBalancer`），则为 `false`。 |
| `s3.uri` | S3 URI (`s3://*<bucket>*/*<key>*`)。 |
| `s3.bucket` | S3 事件的存储桶名称。 |
| `s3.key` | S3 的键名。 |
| `ec2.name` | EC2 实例的名称，通常存储在实例标签中。 |

## Kubernetes 审计日志

关于 Kubernetes 审计日志的字段（列在 表 6-11 中）在配置了 k8saudit 插件时可用。k8saudit 插件负责将 Falco 与 Kubernetes 审计日志设施接口化。插件导出的字段可用于监视多种类型的 Kubernetes 活动。

Table 6-11\. `k8saudit` 过滤器类字段

| Field name | 描述 |
| --- | --- |
| `ka.user.name` | 执行请求的用户名称 |
| `ka.user.groups` | 用户所属的组 |
| `ka.verb` | 正在执行的操作 |
| `ka.uri` | 从客户端发送到服务器的请求 URI |
| `ka.uri.param` | URI 中给定查询参数的值（例如，当 `uri=/foo?key=val` 时，`ka.uri.param[key]` 是 `val`） |
| `ka.target.name` | 目标对象的名称 |
| `ka.target.namespace` | 目标对象的命名空间 |
| `ka.target.resource` | 目标对象的资源 |
| `ka.req.configmap.name` | 当请求对象指向 ConfigMap 时，ConfigMap 的名称 |
| `ka.req.pod.containers.image` | 当请求对象指向 Pod 时，容器的镜像 |
| `ka.req.pod.containers​.privi⁠leged` | 当请求对象指向 Pod 时，所有容器的 `privileged` 标志的值 |
| `ka.req.pod.containers .add_capabilities` | 当请求对象指向 Pod 时，在运行容器时添加的所有能力 |
| `ka.req.role.rules` | 当请求对象指向角色或集群角色时，与角色关联的规则 |
| `ka.req.role.rules.verbs` | 当请求对象指向角色或集群角色时，与角色规则关联的动词 |
| `ka.req.role.rules .resources` | 当请求对象指向角色或集群角色时，与角色规则关联的资源 |
| `ka.req.service.type` | 当请求对象涉及服务时，服务类型 |
| `ka.resp.name` | 响应对象的名称 |
| `ka.response.code` | 响应代码 |
| `ka.response.reason` | 响应原因（通常仅在失败时出现） |

# 结论

恭喜，你现在已经是一个过滤专家了！此时，你应该能够阅读和理解 Falco 规则，并且离能够编写自己的规则更近了一步。在下一章中，我们将专注于 Falco 的输出。
