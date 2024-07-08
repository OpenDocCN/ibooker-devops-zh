# 第十四章：Falco 开发

扩展 Falco 是确保其完全符合您独特需求的最佳方式。本章将展示三种扩展 Falco 的方法。我们将首先概述 Falco 的代码库，并快速指导如何从源代码构建 Falco，这样您可以直接处理 Falco 的代码。这种第一种方法给予您更多自由，但可能比其他两种方法更困难，也许不太方便。第二种方法允许您通过与 gRPC API 交互构建处理 Falco 通知的应用程序。第三种方法是扩展 Falco 的标准且最简单的方式：编写您自己的插件。

对于最后两种方法，我们将通过示例来进行讲解。在这些代码片段中，我们使用 Go 编程语言，因此对其有一些了解将会有所帮助，但不是必需的。本章还假定您已经阅读了本书的第二部分。如果您担心这些材料可能过于困难，不要害怕：我们认为即使您不是开发人员，您也会发现它易于理解且有趣。

# 与代码库一起工作

Falco 是开源的，其所有源代码都存储在 GitHub 上的 Falcosecuriy 组织下。要开始浏览代码库，您只需要一个浏览器。如果您希望将源代码存储在本地并使用您喜欢的编辑器打开它，您需要使用 Git。

Falcosecurity 组织托管了 Falco 和许多其他相关项目。社区非常活跃，因此您还会找到许多实验性项目。Falco 项目的核心存放在两个主要仓库中：*falcosecurity/falco* 和 *falcosecurity/libs*。

## falcosecurity/falco 仓库

[*falcosecurity/falco* 仓库](https://oreil.ly/lqnL4) 包含 *falco* 用户空间程序（通常与您进行交互的程序）的源代码。这是主要且最重要的仓库。该项目组织如下：

*/cmake*

在这里，您可以找到 Falco 构建系统用于拉取依赖项和实现特定功能的 cmake 模块，包括在构建过程中拉取 *falcosecurity/libs* 源代码的 cmake 文件。

*/docker*

此文件夹分为各种子目录，每个子目录包含 Falco 容器镜像的源代码。一些并未发布，因为它们仅供开发使用。详细信息请参阅[README 文件](https://oreil.ly/oiGQQ)。

*/proposals*

此文件夹包含由社区提出并由维护者批准的设计提案。您可能会在这里找到有用的信息，帮助您理解 Falco 作者在某些架构决策及其背后的理由上做出的决定。

*/rules*

默认的规则文件存储在这里。

*/scripts*

此文件夹中包含各种脚本文件。例如，您将在此找到 *falco-driver-loader* 脚本的源代码。

*/test* 和 */tests*

这两个文件夹分别包含 Falco 的回归测试和单元测试。

*/userspace*

Falco 的实际 C++源代码位于这个文件夹中。它的内容被组织成两个子目录：*engine*，其中包含规则引擎实现，以及*falco*，其中包含高层次功能的实现，如输出通道、gRPC 服务器和 CLI 应用程序。

尽管这是 Falco 主仓库，但项目的大部分源代码并不在这里。大部分实际上在*falcosecurity/libs*仓库中，它包含 Falco 的核心低级逻辑的实现。

## falcosecurity/libs 仓库

在本书的整个过程中，我们多次提到了*libscap*、*libsinsp*和驱动程序。[*falcosecurity/libs*仓库](https://oreil.ly/HSLDT)托管这些组件的源代码。它的组织如下：

*/cmake/modules*

此文件夹包含用于拉取外部依赖项的 cmake 模块和*libscap*和*libsinsp*的模块定义，这些消费者应用程序（如 Falco）可以使用。

*/driver*

此文件夹包括内核模块和 eBPF 探针的源代码（主要用 C 编写）。

*/proposals*

与 Falco 仓库中的类似，此文件夹包含设计提案文档。

*/userspace*

组   这里的几个子目录中，你可以找到*libsinsp*和*libscap*的源代码（C 和 C++）以及其他共享代码。

本仓库包含用于内核仪器和数据丰富的所有低级逻辑。过滤语法、插件框架实现和许多其他功能都托管在这里。*libs*代码库庞大，但不要让它吓倒你：理解它所需的只是良好的 C/C++知识。

## 从源构建 Falco

从其源代码编译 Falco 类似于编译其他使用 cmake 的 C++项目。构建系统需要一些依赖项：cmake、make、gcc、wget，当然，还有 git（获取 Falco 仓库的本地副本也需要 Git）。你可以在[文档](https://oreil.ly/UMJI2)中找到如何安装这些依赖项的说明。

一旦确保系统上安装了所需的依赖项，可以使用以下命令获取本地副本：

```
$ git clone git@github.com:falcosecurity/falco.git
```

Git 会将仓库克隆到一个名为*falco*的新创建文件夹中。进入该目录：

```
$ cd falco
```

准备一个目录来包含构建文件，然后进入其中：

```
$ mkdir -p build
$ cd build
```

最后，在构建目录中运行：

```
$ cmake -DUSE_BUNDLED_DEPS=On ..
$ make falco
```

第一次运行此命令可能需要大量时间，因为 cmake 会下载并构建所有依赖项。这是因为我们使用了`-DUSE_BUNDLED_DEPS=On`进行配置；或者，你可以设置`-DUSE_BUNDLED_DEPS=Off`来使用系统依赖项，但如果这样做，你需要在构建 Falco 之前手动安装所有必需的依赖项到你的系统上。你可以在文档中找到更新的依赖项列表和其他有用的 cmake 选项。

当`make`命令完成后，如果没有错误，你应该会在*./userspace/falco/falco*（相对于构建目录的路径）下找到新创建的 Falco 可执行文件。

现在，如果你也想从源代码构建驱动程序，并且你的系统已经安装了内核头文件，请运行：

```
$ make driver
```

此命令默认仅构建内核模块。如果你想构建 eBPF 探针，请使用：

```
$ cmake -DBUILD_BPF=True ..
$ make bpf
```

在两种情况下，你会在*./driver*（相对于构建目录的路径）下找到新构建的驱动程序。

# 使用 gRPC API 扩展 Falco

虽然你可能会考虑直接向代码库引入新功能，但还有更方便的方法。例如，如果你想扩展 Falco 的输出机制，你可以创建一个在 Falco 之上运行并实现你的业务逻辑的程序。特别是，gRPC API 允许你的程序轻松消费 Falco 通知并接收元数据。

本节将使用一个示例程序向你展示如何开始使用 Falco gRPC API 进行开发。为了跟随示例，请确保你的系统上有一个运行的 Falco 实例，并启用了 gRCP 服务器和 gRPC 输出通道（参见第八章）。你将通过 Unix 套接字使用 gRPC，因此请确保已安装并配置了 Falco。

我们在以下示例中使用[*client-go*库](https://oreil.ly/1bSay)，它使得使用 gRPC API 变得简单直接：

```
package main

import (
   "context"
   "fmt"
   "time"

   "github.com/falcosecurity/client-go/pkg/api/outputs" ![1](img/1.png)
   "github.com/falcosecurity/client-go/pkg/client"  ![1](img/1.png)
)

func main() {

   // Set up a connection to Falco via a Unix socket
   c, err := client.NewForConfig(context.Background(), &client.Config{
      UnixSocketPath: "unix:///var/run/falco.sock", ![2](img/2.png)
   })
   if err != nil {
      panic(err)
   }
   defer c.Close()

   // Subscribe to a stream of Falco notifications
   err = c.OutputsWatch(context.Background(), ![3](img/3.png)
      func(res *outputs.Response) error {
         // Put your business logic here
         fmt.Println(res.Output, res.OutputFields) ![4](img/4.png)
         return nil
      }, time.Second)
   if err != nil {
      panic(err)
   }
}
```

![1](img/#code_id_14_1)

我们首先导入[*client-go*库](https://oreil.ly/iQD2m)。

![2](img/#code_id_14_2)

`main`函数建立一个连接（由变量`c`表示）到 Falco 的 gRPC 服务器，通过 Unix 套接字使用默认路径。

![3](img/#code_id_14_3)

连接`c`允许调用`OutputsWatch`函数，订阅通知流并使用回调函数处理任何传入的通知。

![4](img/#code_id_14_4)

这个示例使用一个[匿名函数](https://oreil.ly/f4htn)，它将通知打印到标准输出。在实际应用中，你应该实现自己的业务逻辑来消费 Falco 通知。

使用 gRPC API 来实现与 Falco 交互的程序非常方便直接。如果你需要使 Falco 与其他数据源一起工作，插件系统可能是你需要的。

# 使用插件扩展 Falco

插件是扩展 Falco 的主要方式，我们在整本书中多次提到过它们。简而言之，插件是符合特定 API 的 [共享库](https://oreil.ly/EkUs3)。在 Falco 插件框架中，插件的主要责任是通过连接 Falco 到外部源并生成事件来添加新的数据源，并通过导出字段列表并解码事件数据以在 Falco 需要时生成字段值，从事件中提取数据。

插件包含生成和解释数据的逻辑。这很强大，因为这意味着 Falco 只关心从插件中收集字段值并将它们组合成规则条件。换句话说，Falco 只知道哪些字段可以使用以及如何获取它们的值；其他所有操作都委托给插件。由于这种系统，你可以将 Falco 连接到任何领域。

设计插件时有几个重要方面需要考虑。首先，具有事件源功能的插件隐式定义了事件负载格式（插件返回给框架的序列化原始事件数据）。具有与数据源兼容的字段提取功能的同一插件或其他插件稍后将能够访问负载，并在提取字段时使用。其次，具有字段提取功能的插件显式定义了绑定到数据源的字段。最后，规则依赖于数据源规范，以消耗其期望格式的事件。

由于描述插件开发的每个技术细节都需要一本专门的书，因此在本节中，我们只提供一个教育性示例，展示如何实现一个既可以生成事件又可以提取字段的插件。如需更全面的覆盖范围，请参阅[文档](https://oreil.ly/004ur)。

我们的示例将实现一个插件，该插件从 bash 历史文件中读取（默认位于 *~/.bash_history*）。每当用户在 shell 中输入命令时，bash 都会将该命令行存储下来。当 shell 会话结束时，bash 会将输入的命令行追加到历史文件中。它基本上就是一个日志文件。虽然它没有令人信服的用例，但它是学习如何创建从日志文件生成事件的插件的简单方法。因此，让我们开始用一些 Go 代码来玩乐。

## 准备一个 Go 中的插件

首先，创建一个文件（我们称之为 *myplugin.go*），并导入一堆 Go 包以简化开发。你还将导入[*tail*](https://oreil.ly/BdIXO)，一个模拟[`tail`命令](https://oreil.ly/OWco5)的库（我们的示例使用它从日志文件中读取），以及来自 Falcosecurity 的 [Go 插件 SDK](https://oreil.ly/lnyhl)中的一组包，让你可以实现具有提取器功能的源插件。你必须使用 `main` 包，否则 Go 将不允许你将其编译为共享对象：

```
package main

import (
   "encoding/json"
   "fmt"
   "io"
   "os"
   "time"

   "github.com/hpcloud/tail"

   "github.com/falcosecurity/plugin-sdk-go/pkg/sdk"
   "github.com/falcosecurity/plugin-sdk-go/pkg/sdk/plugins"
   "github.com/falcosecurity/plugin-sdk-go/pkg/sdk/plugins/extractor"
   "github.com/falcosecurity/plugin-sdk-go/pkg/sdk/plugins/source"
)
```

SDK 定义了一组接口，帮助您按照简化且明确定义的模式实现插件。正如您马上将看到的，您需要通过向表示您的插件的一对数据结构添加方法（也称为 [带接收器的函数](https://oreil.ly/t5aAZ) 在 Go 中）来满足这些接口。在幕后，SDK 导出这些方法作为插件框架所需的调用约定函数（或简单的 C 符号）。（如果您需要回顾，请参阅 “Falco 插件”。）

## 插件状态和初始化

SDK 需要一个表示插件及其状态的数据结构。它可以实现各种可组合的接口，但所有类型的插件至少必须实现 `Info` 接口以公开有关插件的一般信息，并实现 `Init` 接口以使用给定的配置字符串初始化插件。

例如，此示例称此数据结构为 `bashPlugin`。您还将定义另一个数据结构（称为 `bashPluginCfg`），表示插件的配置，以在其中存储选项。这不是强制性的，但通常很方便：

```
// bashPluginCfg represents the plugin configuration.
type bashPluginCfg struct {
   Path string
}

// bashPlugin holds the state of the plugin.
type bashPlugin struct {
   plugins.BasePlugin
   config bashPluginCfg
}
```

现在，您将实现第一个公开有关插件一般信息的必需方法：

```
func (b *bashPlugin) Info() *plugins.Info {
   return &plugins.Info{
      ID:          999,
      Name:        "bash",
      Description: "A Plugin that reads from ~/.bash_history",
      Version:     "0.1.0",
      EventSource: "bash",
   }
}
```

###### 提示

所有源插件都需要 `ID` 字段，并且在它们之间必须是唯一的，以确保互操作性。特殊值 `999` 仅保留用于开发目的；如果您打算分发您的插件，您应该在 [插件注册表](https://oreil.ly/7C9n1) 中注册它以获取唯一的 ID。

另一个重要的互操作性字段是 `EventSource`，您可以在其中声明数据源的名称。提取器插件可以使用该值来确定它们是否与数据源兼容。

另一个必需的方法是 `Init`。Falco 仅在加载插件时调用此方法，并传递配置字符串（在 Falco 配置中为插件定义的字符串）。通常，配置字符串是 JSON 格式的。我们的示例首先为我们之前声明的插件配置数据结构 `b.config` 的成员设置默认值。然后，如果给定的 `config` 字符串不为空，函数将 JSON 值解码到 `b.config` 中：

```
func (b *bashPlugin) Init(config string) error {

   // default value
   homeDir, _ := os.UserHomeDir()
   b.config.Path = homeDir + "/.bash_history"

   // skip empty config
   if config == "" {
       return nil
   }

   // else parse the provided config
   return json.Unmarshal([]byte(config), &b.config)
}
```

## 添加事件源功能

特别是对于具有事件源功能的插件，SDK 需要另一个表示 *捕获会话*（事件流）的数据结构。它还需要以下方法：

+   `Open` 用于启动和初始化捕获会话

+   `NextBatch` 用于生成事件

Falco 在初始化后立即调用`Open`。这表示捕获会话的开始。该方法的主要责任是实例化保存会话状态的数据结构（在我们的示例中为`bashInstance`）。具体来说，我们在这里创建一个`*tail.Tail`实例（模仿`tail -f -n 0`的行为）并将其存储在`t`中。然后我们创建一个`bashInstance`实例（可以将`t`分配给它）并返回它：

```
// bashInstance holds the state of the current session.
type bashInstance struct {
   source.BaseInstance
   t      *tail.Tail
   ticker *time.Ticker
}

func (b *bashPlugin) Open(params string) (source.Instance, error) {
   t, err := tail.TailFile(b.config.Path, tail.Config{
      Follow: true,
      Location: &tail.SeekInfo{
         Offset: 0,
         Whence: os.SEEK_END,
      },
   })
   if err != nil {
      return nil, err
   }

   return &bashInstance{
      t:      t,
      ticker: time.NewTicker(time.Millisecond * 30),
   }, nil
}
```

插件系统存储`Open`返回的值，并将其作为源插件的最重要方法的参数传递：`NextBatch`。与其他方法不同，此方法属于会话数据结构（`bashInstance`），而不属于插件数据结构（`bashPlugin`）。在捕获会话期间，Falco 重复调用`NextBatch`，后者又生成一批新事件。批处理的最大大小取决于其底层可重用内存缓冲区的大小。然而，批处理中的事件数可以少于其最大容量；它可以仅包含一个事件，甚至是空的。该方法通常实现源插件的核心业务逻辑，但本例中仅实现了一些简单的逻辑：尝试从`b.t.Lines`通道接收行并将它们添加到批处理中。如果没有，则在一段时间后会超时：

```
func (b *bashInstance) NextBatch(
	bp sdk.PluginState,
	evts sdk.EventWriters,
) (int, error) {
   i := 0
   b.ticker.Reset(time.Millisecond * 30)

   for i < evts.Len() {
      select {
      case line := <-b.t.Lines:
         if line.Err != nil {
            return i, line.Err
         }

         // Add an event to the batch
         evt := evts.Get(i)
         if _, err := evt.Writer().Write([]byte(line.Text)); err != nil {
            return i, err
         }
         i++
      case <-b.ticker.C:
         // Timeout occurred, return early
         return i, sdk.ErrTimeout
      }
   }

   // The batch is full
   return i, nil
}
```

如您所见，SDK 提供了一个`sdk.EventWriters`接口。这自动管理批处理的可重用内存缓冲区，并允许实现者将原始事件负载作为字节序列写入。函数`evts.Len`返回批处理中允许的最大事件数。

事件负载格式的选择由插件作者决定，因为插件 API 允许数据的编码（在我们的示例中，为了简单起见，我们将整行文本作为普通文本存储在负载中）和数据的解码（稍后我们将看到）。这允许您创建可用于规则的字段。选择正确的格式至关重要，因为它不仅对性能有影响，还对与其他插件的兼容性有影响（其他作者可能希望实现与您的事件兼容的提取器插件）。

到目前为止，您已经看到了实现源插件所需的最小方法集。但是，如果我们不添加一种方法来导出字段以用于规则条件和输出，此时插件实际上并不会真正有用。

## 添加字段提取能力

具有字段提取能力的插件可以从事件数据中提取值并导出 Falco 可以使用的字段。一个插件可以只具有事件源功能（在前一节中描述），也可以只具有字段提取功能，或者两者兼有（就像我们的示例插件一样）。具有字段提取能力的插件将在其他插件提供的数据源上工作，而具有两种能力的插件通常只在自己的数据源上工作。然而，机制是相同的，无论数据源如何。SDK 允许您定义以下方法，这些方法在两种情况下都适用：

+   `Fields` 用于声明插件能够提取哪些字段

+   `Extract` 从事件数据中提取给定字段的值

让我们在示例插件中实现这些方法。第一个方法 `Fields` 返回一个 `sdk.FieldEntry` 切片。每个条目包含单个字段的规范。下面的代码告诉 Falco 插件可以提取名为 `shell.command` 的字段（本例仅添加了一个字段）：

```
func (b *bashPlugin) Fields() []sdk.FieldEntry {
   return []sdk.FieldEntry{
      {Type: "string", Name: "shell.command", Display: "Shell command line", 
       Desc: "The command line typed by user in the shell"},
   }
}
```

现在，为了使提取工作正常，我们需要实现 `Extract` 方法，该方法提供实际的业务逻辑来提取字段。该方法接收提取请求（包含所请求字段的标识符）和一个读取器（用于访问事件负载）作为参数。由于本例只有一个字段，实现起来非常简单，它将简单地返回事件负载的所有内容。在实际场景中，您通常会有更多字段和特定的提取逻辑：

```
func (m *bashPlugin) Extract(req sdk.ExtractRequest, evt sdk.EventReader) error {
   bb, err := io.ReadAll(evt.Reader())
   if err != nil {
      return err
   }

   switch req.FieldID() {
   case 0: // shell.command
      req.SetValue(string(bb))
      return nil
   default:
      return fmt.Errorf("unsupported field: %s", req.Field())
   }
}
```

有了字段提取功能，我们的示例插件几乎准备好了。让我们看看如何完成并使用它。

## 完成插件

你已经快完成了。接下来，你将创建插件的一个实例，并在 SDK 中注册其能力。您可以在 Go 初始化阶段通过使用特殊的 [`init` 函数](https://oreil.ly/LDPaK) 来完成这一过程。（不要与 `Init` 方法混淆！）由于我们的示例插件具有源和提取器功能，因此我们必须使用提供的函数通知 SDK 两者的能力：

```
func init() {
   plugins.SetFactory(func() plugins.Plugin {
      p := &bashPlugin{}
      extractor.Register(p)
      source.Register(p)
      return p
   })
}

func main() {}
```

注意空的 `main` 函数。正如您马上会看到的，Go 构建系统需要它来正确构建插件，但它永远不会调用 `main`，所以您可以始终将其保持为空。

使您的代码成为真正的 Go 项目的最后一步是初始化 Go 模块并下载依赖项：

```
$ go mod init *example.com/my/plugin*
$ go mod tidy
```

这些命令分别创建了 *go.mod* 和 *go.sum* 文件。现在，您的插件代码已经准备就绪。是时候编译它，以便您可以在 Falco 中使用它了！

## 构建用 Go 编写的插件

插件是一个共享库（也称为 *共享对象*），具体来说是一个编译文件，它导出了插件框架所需的一组 C 符号。（我们在示例中使用的 SDK 通过使用高级接口隐藏了这些 C 符号，但它们仍然存在于底层。）

Go 编译器有一个特定的命令称为 [`cgo`](https://oreil.ly/sD0aW)，用于创建与 C 代码接口的 Go 包。它允许您编译插件并获得一个共享库文件（一个 *.so* 或 *.dll* 文件）。该命令非常简单。在存放源代码的同一文件夹中运行：

```
$ go build -buildmode=c-shared -o libmyplugin.so
```

该命令创建了 *libmyplugin.so*，您可以在 Falco 中使用它。 （按照惯例，类 Unix 系统中的共享对象文件以 *lib* 开头，并以 *.so* 作为扩展名。）您在 第十章 中了解了插件配置，但以下部分将为您提供一些关于开发时如何使用插件的提示。

## 在开发过程中使用插件

默认情况下，Falco 会在*/usr/share/falco/plugins*中查找已安装的插件。但是，你可以在配置中指定绝对路径，并将插件放在任何你想放置的位置。（在开发过程中这很方便，因为你不需要将插件安装在默认路径下。）我们建议在开发插件的同时构建插件（使用上一节中的命令）。然后，在同一文件夹中，复制*falco.yaml*，根据需要添加你的插件配置，并将`library_path`选项设置为插件的绝对路径。例如：

```
plugins:
  - name: bash
    library_path: /path/to/your/plugin/libmyplugin.so
    init_config: ""

load_plugins: [bash]
```

现在，在使用插件之前，你需要一个与插件提供的数据源匹配的规则文件。（Falco 即使没有规则文件也会加载插件，但你将不会收到任何通知。）你可以在同一文件夹中创建一个规则文件，例如*myplugin_rules.yaml*，并在其中添加如下规则：

```
- rule: Cat in the shell
  desc: Match command lines starting with "cat".
  condition: shell.command startswith "cat "
  output: Cat in shell detected (command=%shell.command)
  priority: DEBUG
  source: bash
```

一旦准备好你定制的*falco.yaml*和*myplugin_rules.yaml*，最后一步就是运行 Falco 并传递这些文件给相应的选项：

```
$ falco -c falco.yaml -r myplugin_rules.yaml
```

完成！在开发过程中以这种方式运行 Falco 插件非常方便，因为它不需要你安装任何文件或干扰本地的 Falco 安装。

###### 小贴士

如果你按照我们示例中的方法构建了插件，并希望触发规则，可以运行：

```
$ bash
$ cat --version
$ exit
```

# 结论

有几种方法可以扩展 Falco。编写插件通常是最佳选择，特别是如果你希望 Falco 能够处理新的数据源以启用新的用例。如果需要与输出进行接口化，gRPC API 可能会对你有所帮助。在极少数情况下，你可能需要直接修改 Falco 核心及其组件。

无论如何，你都需要阅读文档。有时你可能需要学习和理解高级主题。由于 Falco 是开源的且是一个协作项目，你总是有机会与其充满活力的社区联系。与他人分享想法和知识将帮助你更快找到答案。

你可能还会发现其他人有与你完全相同的需求，并愿意帮助你改进或扩展 Falco。这将是为 Falco 项目做贡献的绝佳机会。每个人都可以为 Falco 做贡献。这不仅是一种有益的经验，而且为项目和所有用户（包括你）提供了极大的帮助。想知道如何做？请阅读下一章！
