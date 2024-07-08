# 第十三章：编写 Falco 规则

欢迎来到本书的第 IV 部分！现在你已经了解了 Falco 的基本信息和功能（第 I 部分），理解了其架构的复杂性（第 II 部分），并且在部署和运行它方面已经很专业了（第 III 部分），现在是时候再次提升你的水平了。

本书的最后部分（第十三章至第十五章）讨论的是超越默认内容的内容。你将学习如何根据自己的特定需求定制 Falco，以及如何（如果你愿意的话）将你的改进贡献给项目，使整个社区受益。这是释放你创造力的地方。

我们在本书中已经广泛讨论了规则，特别是在第七章中。但是，当你能够创建自己的规则并将现有规则适应你的环境时，你才能解锁 Falco 的真正力量——这正是我们将在这里向你展示如何做到的。

本章假设你对字段和过滤器有很好的理解（在第六章中介绍），以及规则和规则文件的基础知识（在第七章中介绍）。如果你觉得需要恢复记忆，只需回到那些章节。我们会等你准备好的。

# 定制默认的 Falco 规则

虽然 Falco 的默认规则集是丰富且不断扩展的，但在遇到需要自定义这些规则的情况下并不少见。以下是一些例子：

+   你希望扩展某个规则的范围或增加其覆盖范围。

+   你希望缩减 Falco 加载的规则数量，以降低其 CPU 使用率。

+   你希望通过控制规则的行为或向其添加异常来减少警报噪声。

Falco 提供了一个框架，可以在不必分叉默认规则文件并维护自己的副本的情况下完成这些事情。第七章 教会了你如何替换和追加宏、列表和规则，以及如何禁用规则。这尤为重要，因为正如你在第十章中学到的那样，规则文件加载的顺序很重要，而你可以控制这个顺序。这意味着你可以在初始化链中后加载的单独文件中更改现有规则。

默认的 Falco 配置被设计为利用这种机制，提供了两个可以在不触及默认规则集的情况下定制现有规则的地方。 第一个是*falco_rules.local.yaml*。 这个文件最初是空的，在加载*falco_rules.yaml*之后加载，因此是禁用或修改默认规则集中规则的好地方。 第二个是*/etc/falco/rules.d*。 默认情况下，Falco 会加载此目录中找到的所有规则文件，在加载*falco_rules.yaml*和*falco_rules.local.yaml*后加载。 这使其成为另一个进行自定义的好地方。

# 写作新的 Falco 规则

从本质上讲，编写新规则只是制作条件和输出的问题，因此在概念上它是一个非常简单的过程。 然而，在实践中，需要考虑几个因素。 即兴的规则开发通常导致不完善甚至无法使用的规则。 经验丰富的 Falco 用户倾向于开发自己的规则编写流程，我们建议您也这样做。 最佳流程取决于您的设置、目标环境和喜好，因此我们无法为您提供绝对的处方。 相反，我们将分享我们的做法，希望它能为您提供灵感和指导。

## 我们的规则开发方法

本书作者使用的规则开发方法由九个步骤组成：

1.  复制您想要检测的事件。

1.  捕获事件并将其保存在跟踪文件中。

1.  使用 sysdig 的帮助制作和测试条件过滤器。

1.  使用 sysdig 的帮助来制作和测试输出。

1.  将 sysdig 命令行转换为规则。

1.  在 Falco 中验证规则。

1.  模块化和优化规则。

1.  创建一个回归。

1.  将规则与社区分享。

在接下来的几节中，我们将扩展列表中的每一项，并提供一个真实的例子，引导您完成制作一个新规则的过程，该规则检测试图在*/proc*、*/bin*和*/etc*目录内创建符号链接的尝试^(1)。 这至少是奇怪的行为，并且可能表明可疑活动。 下面是如何应用我们的方法来构建这样一个规则。

### 1\. 复制您想要检测的事件

要创建可靠的规则，几乎不可能没有测试和验证，因此第一步是重新创建规则应该检测到的场景（或场景）。 在这种情况下，您想要检测三个特定目录中的符号链接的创建。 您可以使用终端中的`ln`命令重新创建该场景：

```
$ ln -s ~ /proc/evillink
$ ln -s ~ /bin/evillink
$ ln -s ~ /etc/evillink
```

### 2\. 捕获事件并将其保存在跟踪文件中

现在，您可以使用 sysdig 捕获可疑活动。 （如果您需要 sysdig 和跟踪文件的复习，请返回到“观察系统调用”。） sysdig 允许您使用`-w`命令行标志轻松将活动存储在跟踪文件中。 要查看其工作原理，请在终端中发出以下命令：

```
$ sysdig -w evillinks.scap
```

在另一个终端中，再次运行三个`ln`命令，然后返回第一个终端并使用 Ctrl-C 停止 sysdig。现在，您可以随意检查您的活动跟踪文件多次：

```
$ sysdig -r evillinks.scap
```

您会注意到跟踪文件包含所有主机的活动，而不仅仅是您的`ln`命令。您还会注意到文件相当大。通过在运行捕获时使用过滤器，您可以使其更小并更易于检查：

```
$ sysdig -w evillinks.scap proc.name=ln
```

现在，您拥有一个少于 1 MB 大小的无噪声文件，其中只包含您需要制作规则的特定活动。将触发规则的活动保存在跟踪文件中有几个优点：

+   它只需要复制复杂行为一次。（并非所有可疑行为都像运行`ln`三次那样简单！）

+   它允许您专注于事件并留在单个终端中，而无需多次复制触发规则的命令。

+   它允许您在不同的机器上开发规则。您甚至不需要在行为发生的机器上部署和配置 Falco！如果您希望捕获云容器或边缘设备等“不友好”环境中的行为，这真是太好了。

+   它使您能够以普通用户权限开发规则。

+   它提供了一致性，不仅对于创建规则很有用，而且在完成规则后实施回归测试时也很有用。

### 3\. 使用 sysdig 制作并测试条件过滤器

现在您已经拥有所需的数据，是时候处理条件了。通常，在这个阶段，您会想要回答几个问题：

1.  您需要针对什么类型的系统调用（或系统调用）？当然，并非所有的 Falco 规则都基于系统调用；例如，您可能在使用插件。但总体而言，识别将触发规则的事件类型是首要任务。

1.  一旦确定要解析的事件，您需要检查其参数或参数中的哪些？

sysdig 可以帮助您回答这些问题。使用它来读取和解码捕获文件：

```
$ sysdig -r evillinks.scap
```

在输出文件的末尾就是魔法发生的地方：

```
2313 11:21:22.782601383 1 ln (23859) > symlinkat 
2314 11:21:22.782662611 1 ln (23859) < symlinkat res=0 target=/home/foo 
linkdirfd=-100(AT_FDCWD) linkpath=/etc/evillink
```

我们的系统调用是`symlinkat`。系统调用的[manpage](https://oreil.ly/oW7rT)告诉您它是另一个系统调用`symlink`的变体。您还可以看到`linkpath`参数包含符号链接的文件系统路径。这正是您需要了解的内容，以便制作您的过滤器，应如下所示：

```
(evt.type=symlink or evt.type=symlinkat) and (
  evt.arg.linkpath startswith /proc/ or 
  evt.arg.linkpath startswith /bin/ or 
  evt.arg.linkpath startswith /etc/
)
```

您可以立即利用 sysdig 验证这是正确的过滤器：

```
$ sysdig -r evillinks.scap \
  "(evt.type=symlink or evt.type=symlinkat) and \
   (evt.arg.linkpath startswith /proc/ or \
   evt.arg.linkpath startswith /bin/ or \
   evt.arg.linkpath startswith /etc/)"
438 11:21:13.204948767 2 ln (23814) < symlinkat res=-2(ENOENT) target=/home/foo 
linkdirfd=-100(AT_FDCWD) linkpath=/proc/evillink 
1679 11:21:19.420360948 0 ln (23850) < symlinkat res=0 target=/home/foo 
linkdirfd=-100(AT_FDCWD) linkpath=/bin/evillink 
2314 11:21:22.782662611 1 ln (23859) < symlinkat res=0 target=/home/foo 
linkdirfd=-100(AT_FDCWD) linkpath=/etc/evillink
```

太好了！输出正确显示了三个应触发规则的系统调用。

### 4\. 使用 sysdig 制作并测试输出

sysdig 非常方便，可以帮助你创建规则的输出。特别是 sysdig 的`-p`标志，接收一个与 Falco 兼容的字符串作为输入，并用它打印类似 Falco 的输出到终端，以便于测试规则的输出，知道当规则触发时 Falco 将显示相同的内容。例如，以下看起来是规则的一个很好的输出：

```
a symlink was created in a sensitive directory (link=%evt.arg.linkpath, 
target=%evt.arg.target, cmd=%proc.cmdline)
```

在 sysdig 中与过滤器一起进行测试：

```
$ sysdig -r evillinks.scap \
  -p"a symlink was created in a sensitive directory \
  (link=%evt.arg.linkpath, target=%evt.arg.target, cmd=%proc.cmdline)" \
  "(evt.type=symlink or evt.type=symlinkat) and \
  (evt.arg.linkpath startswith /proc/ or \
  evt.arg.linkpath startswith /bin/ or \
  evt.arg.linkpath startswith /etc/)"
a symlink was created in a sensitive directory (link=/proc/evillink, 
target=/home/foo, cmd=ln -s /home/foo /proc/evillink)
a symlink was created in a sensitive directory (link=/bin/evillink, 
target=/home/foo, cmd=ln -s /home/foo /bin/evillink)
a symlink was created in a sensitive directory (link=/etc/evillink, 
target=/home/foo, cmd=ln -s /home/foo /etc/evillink)
```

注意过滤器和输出条件周围的引号。这可以防止 Shell 因为它们包含的任何字符而混淆。

你的条件和输出看起来非常不错。是时候切换到 Falco 了！

### 5\. 将 sysdig 命令行转换为规则

下一步是将你拥有的内容转换为 Falco 规则。这只是一个复制粘贴的练习，因为你已经知道条件和输出是如何工作的：

```
- rule: Symlink in a Sensitive Directory
  desc: >
    Detect the creation of a symbolic link 
    in a sensitive directory like /etc or /bin.
  condition: > 
    (evt.type=symlink or evt.type=symlinkat) and (
      evt.arg.linkpath startswith /proc/ or 
      evt.arg.linkpath startswith /bin/ or 
      evt.arg.linkpath startswith /etc/)
  output: >
    a symlink was created in a sensitive directory 
    (link=%evt.arg.linkpath, target=%evt.arg.target, cmd=%proc.cmdline)
  priority: WARNING
```

### 6\. 在 Falco 中验证规则

将规则保存在名为*symlink.yaml*的 YAML 文件中。现在在 Falco 中测试它只需使用`-r`标志加载它，然后使用`-e`标志使用捕获文件作为输入：

```
$ falco -r symlink.yaml -e evillinks.scap
2022-02-05T01:09:23+0000: Falco version 0.31.0 (driver version 
319368f1ad778691164d33d59945e00c5752cd27)
2022-02-05T01:09:23+0000: Falco initialized with configuration file 
/etc/falco/falco.yaml
2022-02-05T01:09:23+0000: Loading rules from file symlink.yaml:
2022-02-05T01:09:23+0000: Reading system call events from file: evillinks.scap
2022-02-04T19:21:13.204948767+0000: Warning a symlink was created in a 
sensitive directory (link=/proc/evillink, target=/home/foo, cmd=ln -s /home/foo 
/proc/evillink)
2022-02-04T19:21:19.420360948+0000: Warning a symlink was created in a 
sensitive directory (link=/bin/evillink, target=/home/foo, cmd=ln -s /home/foo 
/bin/evillink)
2022-02-04T19:21:22.782662611+0000: Warning a symlink was created in a 
sensitive directory (link=/etc/evillink, target=/home/foo, cmd=ln -s /home/foo 
/etc/evillink)
Events detected: 3
Rule counts by severity:
   WARNING: 3
Triggered rules by rule name:
   Symlink in a Sensitive Directory: 3
Syscall event drop monitoring:
   - event drop detected: 0 occurrences
   - num times actions taken: 0
```

规则触发了预期次数，并显示了正确的输出。恭喜！

注意，在 Falco 中，你可以利用与 sysdig 创建的相同跟踪文件。`-e`命令行选项告诉 Falco：“从给定文件读取系统调用，而不是使用驱动程序。当你到达文件末尾时，打印摘要并返回。”对于快速迭代非常方便！

### 7\. 模块化和优化规则

你已经有一个工作规则并进行了测试，但还有改进的空间。第 7 步是将规则模块化：

```
- macro: sensitive_sylink_dir
  condition: >
    (evt.arg.linkpath startswith /proc/ or 
     evt.arg.linkpath startswith /bin/ or 
     evt.arg.linkpath startswith /etc/)

- macro: create_symlink
  condition: (evt.type=symlink or evt.type=symlinkat)

- rule: Symlink in a Sensitive Directory
  desc: > 
    Detect the creation of a symbolic link
    in a sensitive directory like /etc or /bin.
  condition:  create_symlink and sensitive_sylink_dir
  output: >
    a symlink was created in a sensitive directory 
    (link=%evt.arg.linkpath, target=%evt.arg.target, cmd=%proc.cmdline)
  priority: WARNING
```

这将条件的检查移入宏中，使条件更短更易读。这很棒，但你还可以做得更好：

```
- list: symlink_syscalls
  items: [symlink, symlinkat]
- list: sensitive_dirs
  items: [/proc/, /bin/, /etc/]

- macro: create_symlink
  condition: (evt.type in (symlink_syscalls))
- macro: sensitive_sylink_dir
  condition: (evt.arg.linkpath pmatch (sensitive_dirs))

- rule: Symlink in a Sensitive Directory
  desc: >
    Detect the creation of a symbolic link 
    in a sensitive directory like /etc or /bin.
  condition:  create_symlink and sensitive_sylink_dir
  output: >
    a symlink was created in a sensitive directory
    (link=%evt.arg.linkpath, target=%evt.arg.target, cmd=%proc.cmdline)
  priority: WARNING
```

在这里所做的是将条件常量移入列表中。这有多个好处。首先，以一种非侵入性的方式使规则易于扩展。如果你想要添加另一个敏感目录，可以通过将相关项目添加到列表或者更好地说，通过创建第二个`symlink_syscalls`列表来添加它。这还让你有机会通过使用`in`和`pmatch`等操作符来优化规则，以便进行高效的多重检查。

### 8\. 创建一个回归测试

当你创建一个新规则时，特别是如果你的目标是将其包含在官方规则集中，你可能希望在将来能够进行测试。例如，你可能希望确保它仍然能在 Falco 的新版本或不同的 Linux 发行版上工作。你还可能希望在压力下测量其性能（如 CPU 利用率）。你在过程开始时创建的捕获文件是回归测试的良好基础。

作为替代方案，Falco 社区创建了一个名为事件生成器的工具（在第二章中提到），用于测试。该工具对于你添加规则操作非常有用，你或其他人可以在任意机器上实时触发规则。该工具可以以灵活的方式重播触发规则的场景，包括多次触发规则和特定频率触发规则。因此，你可以精确测量其 CPU 利用率。你还可以检查在重压下，规则是否会导致 Falco 性能下降到驱动程序开始丢弃系统调用的程度。

讨论事件生成器的全部内容超出了本书的范围，但你可以查看其[GitHub 存储库](https://oreil.ly/jERpD)以了解更多信息。

### 9\. 与社区分享规则

恭喜，你已经完成了全新规则的开发！现在很重要的一点是，Falco 是社区为社区编写的工具。你编写的每一个新规则都可能对许多其他人有价值，因此你应考虑将其贡献给默认规则集。第十五章将教你一切关于向 Falco 贡献的知识。作为 Falco 的维护者和社区成员，我们预先感谢你决定与社区分享的所有规则。

# 写规则时需要记住的事项

现在我们已经掌握了基础知识，让我们讨论一些略微高级但在开发规则时非常重要的概念。

## 优先级

如第七章所述，每个 Falco 规则必须有一个优先级。规则优先级通常与输出一起报告，可以有以下值之一：

+   `EMERGENCY`

+   `ALERT`

+   `CRITICAL`

+   `ERROR`

+   `WARNING`

+   `NOTICE`

+   `INFORMATIONAL`

+   `DEBUG`

选择适当的规则优先级非常重要，因为通常会基于优先级筛选规则。将规则分配过高的优先级可能会导致警报洪水并降低其价值。

官方 Falco 文档关于如何在默认规则集中使用优先级的解释如下：

+   如果规则涉及写入状态（文件系统等），其优先级为`ERROR`。

+   如果规则涉及未经授权的状态读取（读取敏感文件等），其优先级为`WARNING`。

+   如果规则涉及意外行为（在容器中生成意外 shell、打开意外网络连接等），其优先级为`NOTICE`。

+   如果规则涉及违反良好实践（意外的特权容器、挂载敏感文件系统、以 root 身份运行交互式命令），其优先级为`INFORMATIONAL`。

## 噪音

噪音是制定规则时需要考虑的最关键因素之一，也是安全领域中一般复杂话题。检测工具如 Falco 中检测精度和误报生成之间的权衡是一个持续的紧张源。

通常说，唯一没有误报的规则集是没有规则的规则集。完全避免误报是极其困难并且常常是不切实际的目标，但您可以遵循一些指导原则来减少这个问题：

指南 1：测试和验证。

在生产环境中使用规则之前，请确保在尽可能多的环境中广泛测试它（不同的操作系统发行版、内核、容器引擎和编排器）。

指南 2：优先级及基于优先级的过滤是您的好帮手。

避免首次使用`ERROR`或`CRITICAL`作为优先级部署规则。从`DEBUG`或`INFO`开始，观察情况，如果噪音不大，再逐步提升优先级。低优先级规则可以在输出管道的不同阶段轻松过滤，因此不会在半夜唤醒安全运营中心团队。

指南 3：利用标签。

您分配给规则的标签包含在 Falco 的 gRPC 和 JSON 输出中。这意味着您可以使用它们来补充优先级并更加灵活地过滤 Falco 的输出。

指南 4：计划异常。

良好的规则设计考虑到已知和未知的异常情况，以一种可读且模块化的方式，并且可以轻松扩展。

例如，看看默认规则集中的*写入 rpm 数据库以下*规则：

```
- rule: Write below rpm database
  desc: an attempt to write to the rpm database by any non-rpm related program
  condition: >
    fd.name startswith /var/lib/rpm and open_write
    and not rpm_procs
    and not ansible_running_python
    and not python_running_chef
    and not exe_running_docker_save
    and not amazon_linux_running_python_yum
    and not user_known_write_rpm_database_activities
  output: >
    Rpm database opened for writing by a non-rpm program 
    (command=%proc.cmdline file=%fd.name 
    parent=%proc.pname pcmdline=%proc.pcmdline
    container_id=%container.id image=%container.image.repository)
  priority: ERROR
  tags: [filesystem, software_mgmt, mitre_persistence]
```

注意已知异常如何作为宏（`rpm_procs`，`ansible_running_python`等）包含在规则中，但规则还包括一个宏（`user_known_write_rpm_database_activities`），让用户通过覆盖机制添加自己的异常。

## 性能

性能是撰写和部署规则时需要考虑的另一个重要主题，因为 Falco 通常与高频数据源一起运行。当您将 Falco 与诸如内核模块或 eBPF 探针之类的系统调用源一起使用时，您的整个规则集可能需要每秒评估数百万次。在这样的频率下，规则的性能至关重要。

拥有紧凑的规则集绝对是保持 Falco CPU 利用率可控的良好实践，正如您在第十章中学到的那样。然而，同样重要的是确保您创建的每个新规则都针对性能进行了优化。您的规则的开销或多或少与规则条件需要为每个输入事件执行的字段比较数量成正比。因此，您应该期望像这样一个简单条件：

```
proc.name=p1
```

与像这个更复杂的规则相比，将会使用大约 20%的 CPU：

```
proc.name=p1 or proc.name=p2 or proc.name=p3 or proc.name=p4 or proc.name=p5
```

优化规则就是确保在大多数常见情况下，Falco 引擎需要执行尽可能少的比较。

以下是降低规则 CPU 利用率的一些指南：

+   规则应始终从事件类型检查开始（例如`evt.type=open`或`evt.type in (mkdir, mkdirat)`）。Falco 在这方面很智能：它能理解当你的规则仅限于某些事件类型时，并且仅在接收到匹配事件时评估规则。换句话说，如果你的规则以`evt.type=open`开头，Falco 甚至不会为任何非`open`系统调用的事件开始评估它。这是如此有效（和重要！）以至于当规则不包含事件类型检查时，Falco 会发出警告。

+   在你的规则中，包括那些有较高失败概率的激进比较，最好是在早期而不是晚期。Falco 条件类似于编程语言中的`if`语句：它从左到右评估，直到某些条件失败为止。你越早让条件失败，完成工作所需的工作量就越少。尝试找到限制规则范围的简单方法。你能将其限制在特定的进程、文件或容器吗？能够仅应用于特定用户的子集吗？在规则中编码这些限制，尽早实现。

+   复杂的重型规则逻辑应包含在激进比较和限制之后（向右）。例如，长的异常列表应放在规则的末尾。

+   尽可能使用多值运算符，如`in`和`pmatch`，而不是编写多个比较。换句话说，`evt.type in (mkdir, mkdirat)`比`evt.type=mkdir or evt.type=mkdirat`更好。多值运算符经过了大量优化，在值数量增多时效果更好。

+   通常情况下，小而简单是好的。养成尽可能保持简单的习惯。这不仅会加快处理规则的速度，还会确保它们易读且易维护！

## 标记

标记是制定规则的强大工具。它有三个重要用途：灵活过滤 Falco 加载的规则、为其输出添加上下文、支持通知过滤和优先级设置，从而减少噪音。慷慨地使用标记将提高你的 Falco 体验，并确保你充分利用你的规则。

# 结论

这是一章的高强度内容！编写规则是一个要求严格但也可以很有趣和创造性的主题。而且，编写完美的规则以执行令人印象深刻的检测将使你在同事中获得很多赞誉。

^(1) *符号链接*（symlink）一词是*符号连接*（symbolic link）的缩写；在 Unix 中，它表示对另一个文件或目录的引用。
