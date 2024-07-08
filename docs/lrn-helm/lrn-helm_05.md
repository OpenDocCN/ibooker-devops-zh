# 第五章：开发模板

模板是 Helm 图表的核心，它们占据了图表中大部分文件和内容。这些文件存放在*templates*目录中。当您运行`helm install`和`helm upgrade`等命令时，Helm 会渲染这些模板并将它们发送到 Kubernetes。如果使用`helm template`命令，则会将模板渲染并显示为输出（即发送到标准输出）。

模板引擎支持多种构建模板的方式。在简单情况下，您可以用用户提供或*values.yaml*文件中传递的值替换 Kubernetes 清单 YAML 文件中的值。在更复杂的情况下，您可以构建逻辑到模板中，简化图表消费者需要输入的内容。或者您可以构建能够配置应用程序本身的功能。

在本章中，您将学习如何开发模板以及理解模板语法的工作原理。我们还将介绍一些 Helm 添加到模板中的酷炫功能，使您能够与 YAML 一起工作并与 Kubernetes 交互。在此过程中，我们将看一些您可以应用于自己模板的模式。

# 模板语法

Helm 使用 Go 标准库提供的 Go 文本模板引擎。该语法用于`kubectl`（Kubernetes 的命令行应用程序）模板，Hugo（静态站点生成器）以及许多其他使用 Go 构建的应用程序。模板引擎在 Helm 中的设计是为了处理各种类型的文本文件。

您不需要了解 Go 编程语言来开发模板。模板引擎中有一些 Go 特有的东西，但如果您不了解 Go，可以将它们视为模板语言的细微差别。在学习开发模板时，我们会特别指出它们。

## 动作

逻辑、控制结构和数据评估由`{{` 和 `}}` 包裹。这些称为动作。任何不在动作内的内容都会被复制到输出中。

当花括号用于开始和结束动作时，它们可以伴随 `-` 以删除前导或尾随空格。以下示例说明了这一点：

```
{{ "Hello" -}} , {{- "World" }}
```

此生成的输出为“Hello,World.” 从 `-` 一直到下一个非空白字符，侧面的空白已被删除。`-` 和其余动作之间需要有 ASCII 空格。例如，`{{–12}}` 评估为 –12，因为 `-` 被视为数字的一部分而不是括号的一部分。

在动作中，您可以利用各种功能，包括管道、if/else 语句、循环、变量、子模板和函数等。将它们结合使用提供了一种强大的模板编程方式。

## Helm 传递给模板的信息

当 Helm 渲染模板时，它将一个数据对象传递给模板，您可以访问该对象中的信息。在模板中，该对象表示为 `.`（即一个句点）。它被称为点。此对象具有广泛的可用信息。

在 第四章 中，您已经看到 *values.yaml* 文件中的值如何作为 `.Values` 上的属性可用。`.Values` 上的属性完全基于 *values.yaml* 文件中的值以及传递到图表中的值，每个图表的属性都不具有模式，并且因图表而异。

除了值之外，关于发布的信息，正如 第二章 中首次描述的那样，可以作为 `.Release` 的属性访问。此信息包括：

`.Release.Name`

发布名称。

`.Release.Namespace`

包含正在发布到的命名空间。

`.Release.IsInstall`

设置为 `true` 时，表示发布的是一个工作负载。

`.Release.IsUpgrade`

设置为 `true` 时，表示发布是升级或回滚。

`.Release.Service`

列出执行发布的服务。当 Helm 安装图表时，此值设置为 `"Helm"`。不同的应用程序，即构建在 Helm 上的应用程序，可以将此值设置为它们自己的值。

*Chart.yaml* 文件中的信息也可以在 `.Chart` 数据对象中找到。此信息确实遵循 *Chart.yaml* 文件的模式。包括：

`.Chart.Name`

包含图表的名称。

`.Chart.Version`

图表的版本。

`.Chart.AppVersion`

应用程序版本（如果已设置）。

`.Chart.Annotations`

包含注释的键/值列表。

可以在 *Chart.yaml* 文件中的每个属性都可以访问。名称不同之处在于它们在 *Chart.yaml* 中以小写字母开头，但作为 `.Chart` 对象属性时以大写字母开头。

如果要从 *Chart.yaml* 文件传递自定义信息到模板，则需要使用注释。`.Chart` 对象仅包含在模式中的 *Chart.yaml* 文件中的字段。您不能添加新字段以传递它们，但可以将自定义信息添加到注释中。

不同的 Kubernetes 集群可以具有不同的能力。这可能取决于您使用的 Kubernetes 版本或安装的自定义资源定义（CRD）。Helm 作为 `.Capabilities` 的属性提供了有关集群能力的一些数据。Helm 在部署应用程序时会查询您部署的集群以获取此信息。包括：

`.Capabilities.APIVersions`

包含集群中可用的 API 版本和资源类型。您将在稍后学习如何使用此信息。

`.Capabilities.KubeVersion.Version`

完整的 Kubernetes 版本。

`.Capabilities.KubeVersion.Major`

包含主要的 Kubernetes 版本。由于 Kubernetes 的主要版本未增加，因此设置为 `1`。

`.Capabilities.KubeVersion.Minor`

集群中使用的 Kubernetes 的次要版本。

当使用`helm template`时，Helm 不会像对`helm install`或`helm upgrade`那样对集群进行调查。处理`helm template`时提供给模板的能力信息是 Helm 已经知道的符合 Kubernetes 集群的默认信息。Helm 之所以这样工作，是因为`template`命令预期仅用于处理模板，并且以一种不会意外泄露配置的方式进行处理。

图表可以包含自定义文件。例如，您可以通过`ConfigMap`或`Secret`将配置文件作为图表中的文件传递给应用程序。在图表中未列在*.helmignore*文件中的非特殊文件在模板中可以通过`.Files`访问。这不会让您访问模板文件。

传递给模板的最后一个数据片段是关于当前正在执行的模板的详细信息。Helm 传递：

`.Template.Name`

包含模板的命名空间文件路径。例如，在第四章的*anvil*图中，路径将是*anvil/templates/deployment.yaml*。

`.Template.BasePath`

当前图表的*templates*目录的命名空间路径（例如，*anvil/templates*）。

在本章后面，您将了解在某些情况下如何更改`.`的范围。当范围发生变化时，像`.Capabilities.KubeVersion.Minor`这样的属性将无法在该位置访问。当模板执行开始时，`.`被映射到`$`，并且`$`不会改变。即使范围发生变化，`$.Capabilities.KubeVersion.Minor`和其他传入的数据仍然可访问。您会发现在范围发生变化时，通常仅使用`$`。

现在您已经了解了传递到模板的数据，我们将看看如何在模板中使用和操作该数据。

## 流水线

流水线是一系列命令、函数和变量链接在一起。变量的值或函数的输出作为流水线中下一个函数的输入。流水线的最后一个元素的输出是流水线的输出。以下是一个简单流水线的示例：

```
character: {{ .Values.character | default "Sylvester" | quote }}
```

这个流水线分为三部分，每部分由`|`分隔。第一部分是`.Values.character`，这是`character`的计算值。这可以是来自*values.yaml*文件的值，或者在使用`helm install`、`helm upgrade`或`helm template`渲染图表时传入的值。此值作为最后一个参数传递给`default`函数。如果值为空，`default`将使用“Sylvester”的值代替。`default`的输出作为`quote`函数的输入，确保该值用引号包裹。`quote`的输出从操作中返回。

流水线是一个强大的工具，您可以在模板中使用它来转换所需的数据。它们可用于各种目的，从创建强大的转换到防止简单错误。您能在以下 YAML 输出中发现错误吗？

```
id: 12345e2
```

`id`的值看起来像一个字符串，但它不是。唯一的字母是*e*，其余都是数字。包括 Kubernetes 在内的 YAML 解析器将其解释为科学记数法中的数字。这将导致错误。当您从 Git 获取摘要或提交 ID 的缩短版本时，这样的简短字符串是常见的输出。一个简单的解决方法是将值用引号括起来：

```
id: "12345e2"
```

当值被引号包裹时，YAML 解析器将其解释为字符串。这是使用管道末尾的`quote`函数可以修复或避免错误的情况之一。

## 函数模板

在操作和流水线中，您可以使用模板函数。您已经看到了其中一些，包括本章前面描述的`default`和`quote`函数。函数提供了一种将您拥有的数据转换为所需呈现格式或生成不存在数据的方式。

大多数函数由 Helm 提供，并且设计用于在构建图表时提供帮助。这些函数从简单的函数如`indent`和`nindent`（用于缩进输出）到能够访问集群并获取当前资源和资源类型信息的复杂函数，功能齐全。

为了说明函数，我们可以查看图表中常用的一种模式，以提高可读性。当运行`helm create`时，正如您在第四章中看到的，作为图表的一部分创建了一个 Kubernetes `Deployment`模板。`Deployment`模板包括一个用于安全上下文的部分：

```
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
```

###### 提示

从第四章的完整图表读取，在[*https://github.com/Masterminds/learning-helm/tree/main/chapter4/anvil*](https://github.com/Masterminds/learning-helm/tree/main/chapter4/anvil)。

在*values.yaml*文件中，为`podSecurityContext`有一个 YAML 条目。这意味着在`securityContext`的`template`部分传递的确切 YAML。在模板内，*values.yaml*文件中的信息不再是 YAML。而是一个数据对象。`toYaml`函数将数据转换为 YAML 格式。

`securityContext`下的 YAML 需要正确缩进，否则部署清单将由于某个部分未正确缩进而产生 YAML 错误。这通过使用两个函数来实现。在`toYaml`的左边，使用了一个带有`{{`的`-`，以删除直到前一行上的`:`之前的所有空白。`toYaml`的输出传递给`nindent`。此函数在接收到的文本开头添加一个换行符，然后对每一行进行缩进。

为了增强可读性，使用 `nindent` 而不是 `indent` 函数。`indent` 函数不在开头添加换行符。使用 `nindent` 是为了让 `securityContext` 下的 YAML 能够在新行上。这是模板中另一个常见的模式。

###### 提示

除了 `toYaml`，Helm 还有将数据转换为 JSON 的函数 `toJson` 和转换为 TOML 的函数 `toToml`。`toYaml` 经常用于创建 Kubernetes 清单，而 `toJson` 和 `toToml` 更常用于创建通过 `Secret` 和 `ConfigMap` 传递给应用程序的配置文件。

函数中传递参数的顺序是有意义的。当使用管道时，一个函数的输出作为管道中下一个函数的最后一个参数传递。在前面的例子中，`toYaml` 的输出作为 `nindent` 的最后一个参数传递，`nindent` 接受两个参数。函数参数的顺序设计用于常见的管道用例。

在模板中有超过 [百个函数](https://oreil.ly/Xtoya) 可供使用。这些包括处理数学、字典和列表、反射、哈希生成、日期函数等等。

## 方法

到目前为止，您已经看到了模板函数。Helm 还包括检测 Kubernetes 集群能力和处理文件的函数方法。

`.Capabilities` 对象具有方法 `.Capabilities.APIVersions.Has`，接受一个参数用于检查您想要检查的 Kubernetes API 或类型的存在。它返回 true 或 false 来告知您该资源在您的集群中是否可用。您可以检查一个组和版本，例如 `batch/v1` 或资源类型如 `apps/v1/Deployment`。

###### 提示

在处理自定义资源定义和多版本 Kubernetes 资源类型时，检查资源和 API 组的存在非常有用。随着 Kubernetes API 版本从 alpha 到 beta，再到发布版本的变化，您希望使用资源类型的最新版本，因为 alpha 和 beta 将会被废弃并从 Kubernetes 中移除。如果您的应用程序将安装在广泛的 Kubernetes 版本上，支持所有这些集群中的 API 版本将非常有用。

###### 警告

当使用 `helm template` 时，Helm 将使用符合 Kubernetes 集群的默认 API 版本集，而不是与您的集群交互以生成已知能力。

您将找到方法的另一个地方是在 `.Files` 上。它包括以下方法来帮助您处理文件：

`.Files.Get name`

提供了一种将文件内容作为字符串获取的方法。在这种情况下，`name` 是从图表根目录开始的包含文件路径的名称。

`.Files.GetBytes`

与 `.Files.Get` 类似，但返回的是文件作为字节的数组。在 Go 术语中，这是一个字节切片（即 `[]byte`）。

`.Files.Glob`

接受一个 glob 模式，并返回另一个`files`对象，其中仅包含文件名与模式匹配的文件。

`.Files.AsConfig`

接受文件组并将其作为适合包含在 Kubernetes `ConfigMap`清单的`data`部分中的扁平化 YAML 返回。当与`.Files.Glob`配对使用时很有用。

`.Files.AsSecrets`

与`.Files.AsConfig`类似。它不是返回扁平化的 YAML 而是返回可以包含在 Kubernetes `Secret`清单的`data`部分中的数据格式。它是 Base64 编码的。当与`.Files.Glob`配对使用时很有用。例如，`{{ .Files.Glob("mysecrets/**").AsSecrets }}`。

`.Files.Lines`

有一个文件名参数，返回文件内容作为以新行分隔的数组（即`\n`）。

为了说明这些的使用，以下模板来自一个*example*图表。它读取图表的*config*子目录中的所有文件，并将每个文件嵌入一个`Secret`中：

```
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "example.fullname" . }}
type: Opaque
data:
{{ (.Files.Glob "config/*").AsSecrets | indent 2 }}
```

如 Helm 的以下示例输出所示，每个文件可以在其自己的键中找到：

```
apiVersion: v1
kind: Secret
metadata:
  name: myapp
type: Opaque
data:
  jetpack.ini: ZW5hYmxlZCA9IHRydWU=
  rocket.yaml: ZW5hYmxlZDogdHJ1ZQ==
```

## 在图表中查询 Kubernetes 资源

Helm 包含一个模板函数，使您能够在 Kubernetes 集群中查找资源。`lookup`模板函数能够返回单个对象或对象列表。在执行不与集群交互的命令时，此函数返回空响应。

以下示例查找在*anvil*命名空间中名为*runner*的`Deployment`，并使元数据注释可用：

```
{{ (lookup "apps/v1" "Deployment" "anvil" "runner").metadata.annotations }}
```

`lookup`函数传入四个参数：

API 版本

这是任何对象的版本，无论是包含在 Kubernetes 中还是作为插件的一部分安装的。此类示例看起来像`"v1"`和`"apps/v1"`。

对象的种类

这可以是任何资源类型。

要查找对象的命名空间

这可以留空以查找您有权限访问的所有命名空间或全局资源，例如 Namespace。

要查找的资源的名称

这可以留空以返回资源列表而不是特定的一个。

当返回资源列表时，您需要循环遍历结果以访问每个单独对象的数据。当对象的查找返回一个*dict*时，列表对象的查找返回一个*list*。这两种类型是 Helm 在模板中提供的。

当返回列表时，对象位于`items`属性中：

```
{{ (lookup "v1" "ConfigMap" "anvil" "").items }}
```

可以使用循环迭代这些项，您将在本章后面学习。以下示例返回在*anvil*命名空间中的所有`ConfigMap`，假设您有权限访问该命名空间。

在使用此函数时应该小心。例如，当作为干预运行时，它将返回与升级运行时不同的结果。干预运行不会与集群交互，因此此函数将不返回结果。当运行升级时它将返回结果。

安装或升级各个集群时返回的结果也可能不同。例如，在开发环境和生产环境中，安装在集群中的资源将有可能导致不平等的响应。

## if/else/with

Go 模板有`if`和`else`语句，以及类似但略有不同的`with`。`if`和`else`的工作方式与大多数编程语言中的相同。为了说明`if`语句，我们可以查看使用`helm create`命令生成的图表中的模式，该命令在第四章中涵盖了。在该图表中，*values.yaml*文件包含一个关于`ingress`的部分，其中包含一个 enabled 属性。看起来像：

```
ingress:
  enabled: false
```

在创建用于 Kubernetes 的`Ingress`资源的*ingress.yaml*文件中，第一行和最后一行是实现此目的的`if`语句：

```
{{- if .Values.ingress.enabled -}}
...
{{- end }}
```

在这种情况下，`if`语句评估`if`语句后面管道的输出是真还是假。如果为真，则评估内部内容。为了知道块的结尾在哪里，您需要一个`end`语句。这很重要，因为缩进或更典型的括号可能是您想呈现的材料的一部分。

使用`if`语句是通常实现的常见*enabled*模式。

`if`语句可以有一个`else`语句，如果`if`语句评估为 false，则执行该语句。以下示例在*Ingress*未启用时将 YAML 注释打印到输出中：

```
{{- if .Values.ingress.enabled -}}
...
{{- else -}}
# Ingress not enabled
{{- end }}
```

有时，您将希望通过将它们与`and`或`or`语句结合来评估多个元素的`if`语句。在模板中，这与您习惯的方式有所不同。考虑来自模板的以下片段：

```
{{- if and .Values.characters .Values.products -}}
...
{{- end }}
```

在这种情况下，`and`被实现为具有两个参数的函数。这意味着`and`出现在使用的两个项目之前。相同的想法适用于`or`的使用，它也作为一个函数实现。

当要与`and`或`or`一起使用的元素之一是函数或管道时，可以使用括号。以下示例中的一个参数是`or`的相等检查：

```
{{- if or (eq .Values.character "Wile E. Coyote") .Values.products -}}
...
{{- end }}
```

使用`eq`函数实现的相等性检查的输出作为`or`的第一个参数传递。括号使您能够将元素分组在一起以构建更复杂的逻辑。

`with`类似于`if`，但`with`块内的作用域会发生变化。继续使用来自`Ingress`的示例，以下块显示了作用域的更改：

```
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
```

如果传入`with`的值为空，则跳过该块。如果值不为空，则执行块，并且块内的`.`的值是`.Values.ingress.annotations`。在这种情况下，块内的作用域已更改为`with`检查的值。

###### 提示

使用 `with` 检查值并使用 `toYaml` 和 `nindent` 函数将其发送到输出的模式，在 *values.yaml* 文件中是常见的，您希望直接在模板中输出它们。这通常用于镜像拉取密钥、节点选择器等。

与 `if` 语句一样，`with` 可以有一个伴随的 `else` 块，当值为空时可以使用。

## 变量

在模板中，您可以创建自己的变量，并将它们用作函数参数传递、输出打印等。变量以 `$` 开头并且有类型。一旦为一个类型（如字符串）创建了变量，就不能将其值设置为另一种类型（如整数）。

使用 `:=` 创建和初始化变量具有特殊语法，例如以下示例：

```
{{ $var := .Values.character }}
```

在这种情况下，创建一个新变量并将 `.Values.character` 的值分配给它。此变量可以在其他地方使用；例如：

```
character: {{ $var | default "Sylvester" | quote }}
```

`$var` 的值与之前章节中传递 `.Values.character` 的方式相同，传递给 `default`。

创建具有初始值的变量的方法与更改现有变量的值的方法不同。当您为现有变量分配新值时，使用 `=`。例如：

```
{{ $var := .Values.character }}
{{ $var = "Tweety" }}
```

在这种情况下，变量在另一个操作中更改。变量在模板执行的整个生命周期内存在，并且可以在模板的后续相同或不同的操作中使用。

###### 注意

变量处理反映了 Go 编程语言中使用的语法和风格。它通过 `:=`、`=` 和类型来遵循相同的语义。

## 循环

使用循环是简化用户与图表交互的常用方法。例如，您可以使用循环来收集在公开 Web 应用程序时使用的主机列表，并循环遍历该列表以创建更复杂的 Kubernetes `Ingress` 资源。

模板中的循环语法与许多编程语言中的不同。不是使用 `for` 循环，而是使用 `range` 循环来迭代 *dicts*（也称为映射）和列表。

下面的示例说明了 dicts 和 lists：

```
# An example list in YAML
characters:
  - Sylvester
  - Tweety
  - Road Runner
  - Wile E. Coyote

# An example map in YAML
products:
  anvil: They ring like a bell
  grease: 50% slippery
  boomerang: Guaranteed to return
```

您可以将列表视为数组，而映射具有键名和值，类似于 Python 中的字典或 Java 中的 HashMap。在 Helm 模板中，您可以使用 `dict` 和 `list` 函数创建自己的字典和列表。

您可以使用 `range` 函数的两种方式。以下示例遍历 *characters* 并更改范围，即 `.` 的值：

```
characters:
{{- range .Values.characters }}
  - {{ . | quote }}
{{- end }}
```

在这种情况下，`range` 遍历列表中的每个项，并将 `.` 的值设置为 Helm 在迭代项时的每个项的值。在此示例中，该值被传递到管道中的 `quote`。`. `的范围在块中更改到` end`，它作为循环的闭合括号或语句。

此代码段的输出是：

```
characters:
  - "Sylvester"
  - "Tweety"
  - "Road Runner"
  - "Wile E. Coyote"
```

另一种使用`range`的方式是让它为键和值创建新变量。这将适用于列表和字典。下一个示例创建了您可以在块中使用的变量：

```
products:
{{- range $key, $value := .Values.products }}
  - {{ $key }}: {{ $value | quote }}
{{- end }}
```

`$key`变量包含映射或字典中的键和列表中的数字。`$value`包含值。如果这是复杂类型（如另一个字典），那么它将作为`$value`可用。新变量在`range`块的末尾处于作用域中，这由相应的`end`动作表示。此示例的输出是：

```
products:
  - anvil: "They ring like a bell"
  - boomerang: "Guaranteed to return"
  - grease: "50% slippery"
```

# 命名模板

有时候，当您需要在 Kubernetes 清单模板中调用模板时，例如，当您通过一些复杂逻辑生成的值或者当您有一个在众多 Kubernetes 清单中重复的部分时，您可以创建自己的模板。Helm 不会自动渲染这些模板，但可以在 Kubernetes 清单的模板中使用它们。

一个例子是当您运行`helm create`生成图表时。默认情况下，Helm 会创建几个带有共享元素（如标签）的 Kubernetes 清单。为了保持标签的一致性，并且只需在一个地方更新它们，Helm 会生成一个模板，然后每次需要标签时调用该模板。

模板中使用的标签有两种类型。一种是用于高级资源（如`Deployment`）的标签，另一种是用于与更新选择器配对的规范中的标签。这些标签需要有所区别，因为规范和选择器中使用的标签通常是不可变的。这意味着你不希望它们包含诸如应用程序版本之类的元素，因为这些元素可能会随着应用程序升级而变化，但规范和选择器不能更新为新版本。

下面的模板选择包含用于生成生成模板中规范和选择器部分的选择器标签。名称*anvil*来自于第四章生成的图表：

```
{{/*
Selector labels ![1](img/1.png)
*/}}
{{- define "anvil.selectorLabels" -}} ![2](img/2.png)
app.kubernetes.io/name: {{ include "anvil.name" . }} ![3](img/3.png)
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}} ![4](img/4.png)
```

![1](img/#co_developing_templates_CO1-1)

在定义函数之前的注释。动作中的注释以`/*`开头，并以`*/`结尾。

![2](img/#co_developing_templates_CO1-2)

你可以使用`define`语句定义一个模板，后跟模板的名称。

![3](img/#co_developing_templates_CO1-3)

模板的内容就像任何其他模板的内容一样。

![4](img/#co_developing_templates_CO1-4)

通过与`define`语句匹配的`end`语句关闭模板的定义。

此模板包含了一些您应考虑在自己的模板中使用的有用内容：

1.  描述模板的注释。在模板渲染时会被忽略，但在代码注释中同样有用。

1.  名称使用`.`作为分隔符进行命名空间处理，以包含图表名称。在第六章中，您将了解库图表和依赖图表。在模板名称上使用命名空间使得能够使用库图表，并避免在依赖图表上发生冲突。

1.  `define`和`end`调用使用操作来移除它们之前和之后的空格，以便它们的使用不会向最终输出的 YAML 添加额外的行。

此模板在资源的`spec`部分被称为，例如*anvil*图表中的`Deployment`：

```
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "anvil.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "anvil.selectorLabels" . | nindent 8 }}
```

此处的`matchLabels`部分是不可变的，因此不能更改，并且它查找`template`部分中的`labels`。

有两个函数可用于在您的模板中包含另一个模板。`template`函数是一个基本函数，用于包含另一个模板。它不能在管道中使用。然后是`include`函数，以类似的方式工作，但可以在管道中使用。在上述示例中，`include`用于调用另一个模板，并将该模板的输出传递给`nindent`，以确保输出具有正确的缩进级别。由于每次调用的输出具有不同的缩进级别，因此缩进级别不能作为定义其自身的模板的一部分。

`include`函数接受两个参数。第一个是要调用的模板的名称。这需要是完整的名称，包括任何命名空间。第二个是要传递的数据对象。这可以是您自己创建的对象，使用`dict`函数，或者它可以是模板内部使用的全局对象的全部或部分。在这种情况下，传递整个全局对象。

Helm 创建的模板函数用于生成更广泛的标签选择，用于标签为可变的高级资源的标签。它具有调用其他用户定义模板的用户定义模板：

```
{{/*
Common labels
*/}}
{{- define "anvil.labels" -}}
helm.sh/chart: {{ include "anvil.chart" . }}
{{ include "anvil.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

因为这些标签是可变的，因此包含了这里将因各种原因而更改的有用标签。为了不重复用于选择器的标签，在这里也包括通过调用生成它们的函数来包含这些标签。

另一种可能遇到的情况是，命名模板很有用的情况，是当您希望封装复杂逻辑时。为了说明这个想法，考虑一个图表，您希望能够将容器版本作为标签、摘要或者回退到应用程序版本作为默认值传递。接受包含容器图像的 Pod 规范的部分是一个单独的行。为了提供这三个选项，您需要许多行逻辑：

```
{{- define "anvil.getImage" -}}
{{- if .Values.image.digest -}}
{{ .Values.image.repository }}@{{ .Values.image.digest }}
{{- else -}}
{{ .Values.image.repository }}:
{{- .Values.image.tag | default .Chart.AppVersion }}
{{- end -}}
{{- end -}}
```

这个新的`getImage`模板能够处理摘要、标签，并且如果前两者都不存在，默认使用应用程序版本。首先会检查并使用摘要。摘要是不可变的，是指定要使用的镜像修订版本的最精确方法。如果没有传入摘要，则会检查标签。标签是指向摘要的指针，可以更改。如果找不到标签，则使用`AppVersion`作为标签。

此函数针对*anvil*图表的结构，最初是为第四章创建的。预期图像细节位于该图表及其*values.yaml*文件的结构中。

在`Deployment`的模板中，使用新函数引用镜像：

```
image: "{{ include "anvil.getImage" . }}"
```

模板可以像软件程序中的函数一样工作。这是一种您可以拆分复杂逻辑和共享功能的有用方式。

# 为了保持模板的可维护性

在*templates*目录中，对模板的强制结构有限。多个 Kubernetes 清单可以位于同一 YAML 文件中，这意味着多个 Kubernetes 清单的模板也可以位于同一文件中。命名模板可以存在于任何模板文件中，并且可以在其他文件中引用。*NOTES.txt*模板是向用户显示的特殊文件，并且测试有特殊处理方式。测试在第六章中进行了覆盖。除此之外，这是一个空白画布，供您创建模板。

为了帮助创建易于维护且易于导航的模板，Helm 维护者建议采用几种模式。这些模式之所以有用，有以下几个原因：

+   您可能长时间不会对图表中的模板进行结构性更改，然后再回来。能够快速重新发现布局将加快流程。

+   其他人将查看图表中的模板。这些可能是创建图表的团队成员或使用它的人。消费者有时可能会打开图表以检查它，以便在安装之前或作为复制的一部分进行操作。

+   在调试图表时（下一节会讲到），如果模板有一定结构，会更容易进行调试。

第一个模式是每个 Kubernetes 清单应该在其自己的模板文件中，并且该文件应该有一个描述性的名称。例如，如果只有一个部署，则命名模板为*deployment.yaml*。如果有多个相同类型的清单，例如使用主数据库和副本部署时，可以使用类似*statefulset-primary.yaml*和*statefulset-replica.yaml*的命名。

第二个指导原则是将命名模板放入您自己模板的文件中，命名为 *_helpers.tpl*。 因为这些本质上是您其他模板的辅助模板，所以名称很描述性。 如前所述，名称开头的 _ 会导致它在目录列表的顶部显示，因此您可以在模板中轻松找到它。

当您使用 `helm create` 命令启动一个新的图表时，默认情况下，模板的内容已经遵循这些模式。

# 调试模板

在开发模板时调试模板非常有用。 Helm 提供了三个功能，您可以在开发工作流中使用这些功能来查找问题。 这些功能是除了测试之外，测试在第六章中介绍。

## 干冷运行

安装、升级、回滚和卸载 Helm 图表的命令都有一个标志来启动干冷运行，并模拟过程但不完全执行该过程。 这是通过这些命令上的 `--dry-run` 标志来实现的。 例如，如果您在 *anvil* 图表上的 `install` 命令上使用 `--dry-run` 标志，则可以使用命令 `helm install myanvil anvil --dry-run`。 Helm 将渲染模板，检查模板以确保发送到 Kubernetes 的内容格式正确，并将其发送到输出。 输出看起来类似于正常安装时的输出，但会有两个额外的部分：

```
NAME: myanvil
LAST DEPLOYED: Tue Jun  9 06:58:58 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
HOOKS:
...
MANIFEST:
...
NOTES:
1\. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default ↵
    -l "app.kubernetes.io/name=anvil,app.kubernetes.io/instance=myanvil" ↵
    -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

两个新部分是 *HOOKS* 和 *MANIFEST* 部分，它们将包含 YAML Helm 通常会传递给 Kubernetes 的内容。 而不是发送到输出。 为了简洁起见，不包含完整生成的 YAML，因为这将非常长。

如果模板中有问题，响应将会大不相同。 为了说明这一点，请尝试从 *anvil* 图表的 *deployment.yaml* 文件中删除第一个 `}`，然后再次执行干冷运行安装。 删除 `}` 将导致解析模板中的操作时出错。 Helm 不会输出状态，而会输出如下错误：

```
Error: parse error at (anvil/templates/deployment.yaml:4): unexpected "}" in
operand
```

此处提供了一个提示，用于查找问题所在的地方。 它包括：

+   出错的文件。 *anvil/templates/deployment.yaml*，在这种情况下。

+   文件中发生错误的行号。 在这里是第 4 行。

+   一个关于问题的提示错误消息。 错误消息通常不会显示问题所在，而是显示解析器出现问题的地方。 在这种情况下，一个单独的 `}` 是意外的。

Helm 将检查模板语法中的错误超过错误。 它还将检查输出语法的语法。 为了说明这一点，在相同的 *deployment.yaml* 文件中删除开头的 `apiVersion:`。 确保补充缺失的 `}`，以修复操作。 现在文件的开头看起来像：

```
apps/v1
kind: Deployment
```

执行干冷运行安装将产生以下输出：

```
Error: YAML parse error on anvil/templates/deployment.yaml: error converting
YAML to JSON: yaml: line 2: mapping values are not allowed in this context
```

也许你想知道为什么在 YAML 和 JSON 之间转换时会出错。这是 Helm 和 Kubernetes 使用的 YAML 解析库的产物。错误消息中有用的部分是从`line 2`开始的部分。第一行不完整，因此第二行处于错误的上下文中，尽管它格式良好。文件不是有效的 YAML，Helm 告诉你从哪里开始查找问题。如果你将同样的 YAML 部分在在线 YAML 验证器中测试，你会得到相同的错误。

Helm 也能够验证 Kubernetes 资源的模式。这是因为 Kubernetes 为其清单提供了模式定义。为了说明这一点，在*deployment.yaml*中将`apiVersion`更改为`foo`：

```
foo: apps/v1
kind: Deployment
```

执行 dry-run 安装将产生以下输出：

```
Error: unable to build kubernetes objects from release manifest: error
validating "": error validating data: apiVersion not set
```

部署不再有效，Helm 能够提供关于缺失内容的具体反馈。在这种情况下，`apiVersion`属性未设置。

使用 dry-run 并不是你获取此功能的唯一方式。`helm template`命令提供了类似的体验，但没有完整的调试功能集。`template`命令会将`template`命令转换为 YAML。在此阶段，如果生成的 YAML 无法解析，它会提供一个错误。但`template`命令不会根据 Kubernetes 模式验证 YAML。当使用`template`命令时，Helm 不会警告您如果`apiVersion`被转换为`foo`。这是因为在使用`template`命令时，Helm 不会与 Kubernetes 集群通信以获取验证模式。

## 获取已安装的清单

有时候你在集群中安装一个应用程序，然后其他东西会在之后改变清单。这导致你声明的内容与实际运行的内容之间存在差异。一个例子是服务网格自动向由你的 Helm 图表创建的`Pod`添加一个 Sidecar 容器。

您可以使用`helm get manifest`命令获取由 Helm 部署的原始清单。该命令将检索释放时 Helm 安装的清单。它能够为历史中仍然可用的任何版本的释放检索此信息，如使用`helm history`命令查找的那样。

继续*myanvil*示例，要检索此*anvil*图表实例的清单，您可以运行：

```
$ helm get manifest myanvil
```

输出将包括所有清单，每个新清单的开头都有`---`。以下是输出的前 15 行：

```
---
# Source: anvil/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myanvil-anvil
  labels:
    helm.sh/chart: anvil-0.1.0
    app.kubernetes.io/name: anvil
    app.kubernetes.io/instance: myanvil
    app.kubernetes.io/version: "9.17.49"
    app.kubernetes.io/managed-by: Helm
---
# Source: anvil/templates/service.yaml
apiVersion: v1
kind: Service
...
```

`---`用作 YAML 文档之间的分隔符。除此之外，Helm 还会添加一个 YAML 注释，注明生成清单所使用的源模板。

## 检查图表

您将遇到的一些问题不会显示为 API 规范的违规，并且也不是模板中的问题。例如，Kubernetes 资源需要具有可用作域名的名称。这限制了名称中可以使用的字符及其长度。Kubernetes 提供的 OpenAPI 模式未提供足够的信息来检测发送到 Kubernetes 时将失败的名称。前文在 第四章 中提到的 *lint* 命令能够检测到此类问题，并告诉您问题出现的位置。

为了说明这一点，您可以修改 *anvil* 图表，在 *deployment.yaml* 中将 `Wile` 添加到部署名称的末尾：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "anvil.fullname" . }}-Wile
```

运行 `helm lint anvil` 将生成一个错误，告知您存在的问题：

```
$ helm lint anvil
==> Linting anvil
[ERROR] templates/deployment.yaml: object name does not conform to Kubernetes
naming requirements: "test-release-anvil-Wile"

Error: 1 chart(s) linted, 1 chart(s) failed
```

在这种情况下，`helm lint` 正在指向一个问题，并告诉您它发生的地方。

# 结论

在图表中包含的模板能够强大地在 Kubernetes 内创建资源。这类似于模板周围的编程语言。模板系统具有逻辑、内置函数、自定义模板和调试功能。这意味着您可以通过值收集所需的输入，并生成所需的 Kubernetes 清单。

这些图表还包括依赖项、测试、值文件模式等更多内容。第六章将详细介绍图表的功能和应用。
