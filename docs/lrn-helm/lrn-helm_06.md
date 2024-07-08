# 第六章：高级图表特性

图表不仅仅是关于图表的元数据和一组模板。图表可以有依赖关系，值可以有模式，Helm 具有生命周期钩子，您可以签名图表，等等。在本章中，您将学习有关图表的其他元素，超越模板。

这些功能提供了解决构建软件包时常见问题的强大解决方案。本章从覆盖依赖关系开始。依赖关系是几乎每个软件包管理解决方案的关键部分，因为它们让您在解决方案中利用现有的软件包并建立在他人的工作之上。然后它继续覆盖模式和验证，当您希望帮助图表用户在覆盖 Helm 执行自定义操作的过程中避免问题时，这些都是有用的。本章还涵盖了测试和测试——在开发中测试非常重要，因为它们确保您的软件按预期运行。Helm 提供了安全功能，有助于减轻一些常见的威胁路径，接下来将对其进行覆盖。本章最后讨论了如何使用图表来扩展 Kubernetes API。

在整个本章中，您将看到图表作为您可以参考的示例，位于[*https://github.com/masterminds/learning-helm/blob/main/chapter6*](https://github.com/masterminds/learning-helm/blob/main/chapter6)。它们展示了本章涵盖的不同特性以及 Helm 仓库。

# 图表依赖项。

依赖关系是包管理器及其软件包的常见元素。图表可以依赖于其他图表。这使得在图表中封装服务、重用图表以及将多个图表一起使用成为可能。

为了说明依赖关系，考虑一个安装 WordPress 的图表，这是一款流行的博客软件。WordPress 依赖于一个符合 MySQL 标准的数据库，用于存储博客内容、用户和其他配置。一个符合 MySQL 标准的数据库可以被其他应用程序使用，并且可以作为一个服务来消费。处理 WordPress 中使用 MySQL 的一种方式是将其清单放在 WordPress 图表中。另一种处理方式是有一个独立的 MySQL 图表，而 WordPress 图表则依赖于它。将符合 MySQL 标准的数据库作为独立图表使其能够被多个应用程序使用，并且可以独立构建和测试。

依赖关系在*Chart.yaml*文件中指定。以下是名为*rocket*的图表的*Chart.yaml*文件中的`dependencies`部分。

```
dependencies:
  - name: booster ![1](img/1.png)
    version: ¹.0.0 ![2](img/2.png)
    repository: https://raw.githubusercontent.com/Masterminds/learning-helm/main/
      chapter6/repository/ ![3](img/3.png)
```

![1](img/#co_advanced_chart_features_CO1-1)

仓库中依赖图表的名称。

![2](img/#co_advanced_chart_features_CO1-2)

图表的版本范围字符串。

![3](img/#co_advanced_chart_features_CO1-3)

从仓库中检索图表的位置。

Helm 图表使用语义版本作为它们的版本方案。用于依赖项的`version`字段接受版本范围，有一些用于这些范围的简写语法。例如，`¹.2.3`是`>= 1.2.3, < 2.0.0`的简写。Helm 支持包括`=`, `!=`, `<`, `⇐`, `>`, `>=`, `^`, `~`和`-`在内的范围。不同的范围可以使用空格或逗号组合在一起，以支持逻辑*and*组合，`|`以支持逻辑*or*组合。Helm 还支持使用`X`或`*`作为通配符字符。如果省略版本的一部分，例如省略补丁部分，Helm 将假定缺失部分是通配符。

范围是指定所需版本的首选方法。马上您将学习如何锁定到指定范围内的特定依赖项版本。通过指定范围，可以使用 Helm 命令自动更新到该范围内的最新版本。如果您想要引入修复 bug 或安全更新依赖项，则这非常有用。

`repository`字段是您指定的图表存储库位置，用于从中提取依赖项。您可以通过以下两种方式之一指定此字段：

+   Helm 存储库的 URL。

+   要使用`helm repo add`命令设置的存储库名称。这个名称需要在@之前加上引号（例如，`"@myrepo"`）。

通常使用完整 URL 来指定位置。这将确保在使用图表的每个环境中检索相同的依赖项。

一旦您指定了具有所需版本范围的依赖项，您需要使用 Helm 将这些依赖项锁定到特定版本并检索它们。如果您打算将您的图表打包成图表存档，如第四章中所述，您需要在打包之前锁定并获取依赖项。

解析指定范围内依赖项的最新版本并检索它，您可以使用以下命令：

```
$ helm dependency update .
```

运行该命令后，您将看到以下输出：

```
Saving 1 charts
Downloading booster from repo https://raw.githubusercontent.com/Masterminds/
  learning-helm/main/chapter6/repository/
Deleting outdated charts
```

运行此命令会导致执行几个步骤。

首先，Helm 解析了*booster*图表的最新版本。它使用存储库中的元数据来了解可用的图表版本。从元数据和指定的版本范围中，Helm 找到了最佳匹配。

将解析后的信息写入*Chart.lock*文件。*Chart.lock*文件中不再是版本范围，而是包含要使用的依赖项的具体版本。这对于可重复性非常重要。*Chart.lock*文件由 Helm 管理。用户的更改将在下次运行`helm dep up`（简写语法）时被覆盖。这类似于其他平台上依赖管理器的锁文件。

一旦 Helm 知道要使用的特定版本，它将下载依赖的图表并将其放入 *charts* 子目录中。依赖的图表放置在 *charts* 目录中非常重要，因为这是 Helm 获取它们内容以渲染模板的地方。图表可以以其存档或目录形式存在于 *charts* 目录中。当 Helm 从存储库下载它们时，它们以存档形式存储。

如果您有一个 *Chart.lock* 文件但 *charts* 目录中没有内容，则可以通过运行命令 `helm dependency build` 重新构建 *charts* 目录。这将使用锁定文件以其已确定版本检索依赖项。

一旦您有了依赖项，当您运行诸如 `helm install` 或 `helm upgrade` 的命令时，Helm 将渲染它们的资源。

当您指定依赖项时，您可能还希望将配置从父或主图表传递到依赖图表。如果我们回顾 WordPress 的示例，这可以用于设置要使用的数据库名称。Helm 提供了一种方法，在父图表的值中执行此操作。

在主图表的 *values.yaml* 文件中，您可以创建一个以依赖图表名称命名的新部分。在此部分中，您可以设置要传递的值。您只需设置您想要更改的部分，因为包含在 *values.yaml* 文件中的依赖图表将作为默认值。

在 *rocket* 图表的 *values.yaml* 文件中，有一个部分如下所示：

```
booster:
  image:
    tag: 9.17.49
```

Helm 知道此部分是用于 *booster* 图表。在这种情况下，它将图像标签设置为特定值。可以以这种方式设置依赖图表中的任何值。当运行诸如 `helm install` 的命令时，您可以使用标志设置依赖项的值（例如 `--set`）以及主图表的值。

如果您在同一个图表上有两个依赖项，可以选择在 *Chart.yaml* 文件中使用 `alias` 属性。此属性放置在您想要使用替代名称的每个依赖项旁边，例如 `name`、`version` 和其他属性。通过 `alias`，您可以为每个依赖项提供一个唯一的名称，在其他地方引用它们，例如 *values.yaml* 文件中。

## 条件标志以启用依赖项

Helm 提供了通过配置启用或禁用依赖项的功能。为了说明这个想法，考虑这样一个情况：您希望提供一个 WordPress 博客解决方案，但是给安装 WordPress 的人员选择是否使用作为服务的数据库或包含的数据库的选项。如果安装图表的人选择使用数据库作为服务，则他们将提供该服务的 URL，并且不需要安装数据库。可以通过两种不同的方式在配置中实现这一点。

当您希望通过依赖项控制单个功能的启用或禁用时，可以在依赖项上使用`condition`属性。为了说明这一点，我们将查看*Chart.yaml*文件中条件图表的`dependencies`部分：

```
dependencies:
  - name: booster
    version: ¹.0.0
    condition: booster.enabled
    repository: https://raw.githubusercontent.com/Masterminds/learning-helm/main/
      chapter6/repository/
```

依赖项具有一个`condition`键，其值告诉 Helm 在值中查找以确定是否应启用或禁用它。在*values.yaml*文件中，相应的部分如下所示：

```
booster:
  enabled: false
```

在这种情况下，默认值是禁用依赖项。当某人安装图表时，可以通过传递一个值来启用该依赖项。

当您有多个涉及依赖关系的要启用或禁用的功能时，可以使用`tags`属性。像`condition`一样，这个属性在描述依赖关系时与`name`和`version`并列。它包含一个依赖项的标签列表。为了说明这一点，我们可以看一下另一个名为*tag*的图表的依赖项：

```
dependencies:
  - name: booster
    tags:
      - faster
    version: ¹.0.0
    repository: https://raw.githubusercontent.com/Masterminds/learning-helm/main/
      chapter6/repository/
  - name: rocket
    tags:
      - faster
    version: ¹.0.0
    repository: https://raw.githubusercontent.com/Masterminds/learning-helm/main/
      chapter6/repository/
```

在这里，您将看到两个带有`tags`部分的依赖项。标签是相关标签的列表。在图表的*values.yaml*文件中，您使用`tags`属性：

```
tags:
  faster: false
```

`tags`是具有特殊含义的属性。这里的值告诉 Helm 默认情况下禁用具有标签*faster*的依赖项。当图表的用户在安装或升级时传递一个 true 值时，它们可以被启用。

## 从子图表导入到父图表的值

有时您可能希望将子图表的值导入或拉到父图表中。Helm 提供了两种方法来做到这一点。一种是在子图表明确将一个值导出以供父图表导入的情况下，另一种是在子图表未导出值的情况下。

### 导出属性

`exports`属性是*values.yaml*文件中的一个特殊顶层属性。当子图表声明了一个`export`属性时，其内容可以直接导入到父图表中。

例如，考虑来自子图表*values.yaml*文件的以下内容：

```
exports:
  types:
    foghorn: rooster
```

当父图表声明子图表为依赖项时，可以从导出中导入值，如下所示：

```
dependencies:
  - name: example-child
    version: ¹.0.0
    repository: https://charts.example.com/
    import-values:
      - types
```

在父计算的值中，现在可以在顶层访问这些类型。在 YAML 中，这相当于：

```
foghorn: rooster
```

### 子父格式

当父图表想从子图表导入一个值，但子图表没有导出该值时，有一种方法告诉 Helm 将子值拉入到父图表中。

为了说明这一点，请考虑一个子图表，在其*values.yaml*文件中指定了以下值：

```
types:
  foghorn: rooster
```

这些值没有被导出，但父图表仍然可以导入它们。当在父级中声明依赖项时，可以使用`child`和`parent`文件导入这些值，如以下示例：

```
dependencies:
  - name: example-child
    version: ¹.0.0
    repository: https://charts.example.com/
    import-values:
      - child: types
        parent: characters
```

在导入的两种方法中，使用的是`import-values`属性。Helm 知道如何区分不同的格式，您可以混合使用这两种格式。

在子图表中，当 `types` 的顶级属性在其计算的值中不可用于父图表中的 `characters` 的顶级属性时，可以表示为 YAML：

```
characters:
  foghorn: rooster
```

此格式允许访问嵌套值，除了使用点作为分隔符的顶级属性。例如，如果子图表具有以下格式，则 `import-values` 上的 `child` 属性可以读取 `data.types`：

```
data:
  types:
    foghorn: rooster
```

# 库图表

您可能会遇到创建多个相似图表的情况——这些图表共享许多相同的模板。对于这些情况，有库图表。

库图表在概念上类似于软件库。它们提供可重用的功能，可以被其他图表导入和使用，但本身不能被安装。

如果您使用 `helm create` 创建一个新的库图表，第一步是删除 *templates* 目录和 *values.yaml* 文件的内容，因为这两者都不会被使用。然后，您需要告诉 Helm 这是一个库图表。在 *Chart.yaml* 文件中将 `type` 设置为 `library`。为了说明这一点，这里是名为 *mylib* 的图表的 *Chart.yaml* 文件：

```
apiVersion: v2
name: mylib
type: library
description: an example library chart
version: 0.1.0
```

当未设置时，`type` 的默认值为应用程序。仅当您的图表是库时才需要设置它。

在 *templates* 目录中以下划线（即 `` `_` ``）开头的文件不应该渲染为发送到 Kubernetes 的清单。约定是帮助模板和片段在 _**.tpl* 和 _**.yaml* 文件中。

为了说明可重用模板的工作原理，以下是在 *mylib* 图表文件中创建 `ConfigMap` 的模板，命名为 *_configmap.yaml*：

```
{{- define "mylib.configmap.tpl" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mylib.fullname" . }} ![1](img/1.png)
  labels:
    {{- include "mylib.labels" . | nindent 4 }} ![2](img/2.png)
data: {}
{{- end -}}
{{- define "mylib.configmap" -}} ![3](img/3.png)
{{- template "mylib.util.merge" (append . "mylib.configmap.tpl") -}}
{{- end -}}
```

![1](img/#co_advanced_chart_features_CO2-1)

`fullname` 函数与 `helm create` 生成的函数相同。

![2](img/#co_advanced_chart_features_CO2-2)

`labels` 函数生成 Helm 推荐在图表中使用的常见标签。

![3](img/#co_advanced_chart_features_CO2-3)

定义了一个特殊的模板，知道如何合并模板在一起。

大部分此定义看起来类似于你会放入 *templates* 目录中的其他模板。 `define` 是一个函数，用于定义在其他地方使用的模板。此文件中定义了两个模板。 *mylib.configmap.tpl* 包含一个资源的模板。这将看起来类似于其他模板。它提供了一个蓝图，旨在被包含此库的图表中的调用者覆盖。 *mylib.configmap* 是一个特殊的模板。这是另一个图表将使用的模板。它将 *mylib.configmap.tpl* 与另一个尚未定义的模板（包含覆盖）合并为一个输出。 *mylib.configmap* 使用一个实用函数来处理合并，并方便重用。该函数是：

```
{{- /*
mylib.util.merge will merge two YAML templates and output the result.
This takes an array of three values:
- the top context
- the template name of the overrides (destination)
- the template name of the base (source)
*/ -}}
{{- define "mylib.util.merge" -}}
{{- $top := first . -}}
{{- $overrides := fromYaml (include (index . 1) $top) | default (dict ) -}}
{{- $tpl := fromYaml (include (index . 2) $top) | default (dict ) -}}
{{- toYaml (merge $overrides $tpl) -}}
{{- end -}}
```

此函数接受一个上下文（考虑在第五章中涵盖的`.`数据），一个包含覆盖项的模板以及要覆盖的基础模板函数。当您看到它的使用方式时，这个函数将变得更加清晰。

###### 注意

在官方将库图表包含在 Helm 中之前，已经开发了库图表的概念。`merge`函数由 Adnan Abdulhussein 开发，作为其通过一个名为*Common*的图表开发这一概念的一部分。

为了说明使用此库函数，以下模板来自另一个名为*mychart*的图表。在使用其定义的资源之前，需要将其作为依赖项添加，就像任何其他依赖项一样。*mychart*中包含一个模板用于创建`ConfigMap`：

```
{{- include "mylib.configmap" (list . "mychart.configmap") -}} ![1](img/1.png)
{{- define "mychart.configmap" -}} ![2](img/2.png)
data: ![3](img/3.png)
  myvalue: "Hello Bosko"
{{- end -}}
```

![1](img/#co_advanced_chart_features_CO3-1)

包括并使用库图表的功能来生成`ConfigMap`。

![2](img/#co_advanced_chart_features_CO3-2)

定义了一个新模板，其中仅包含要覆盖库提供的模板的部分。

![3](img/#co_advanced_chart_features_CO3-3)

提供`data`部分供`ConfigMap`使用。

这个模板一开始可能看起来令人困惑，因为其中涉及的内容很多。

第一行包括来自库图表的`ConfigMap`模板。将一个新列表传递给它，其中包含两个条目。第一个是当前的数据对象，第二个是另一个模板的名称，该模板包含要覆盖库图表提供的元素。

文件的其余部分是包含覆盖项的模板。在库图表提供的模板中，`data`部分未提供任何内容。它是空的。函数`mychart.configmap`提供了一个`data`部分。

此模板的 Helm 渲染输出为：

```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/instance: example
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: mychart
    helm.sh/chart: mychart-0.1.0
  name: example-mychart
data:
  myvalue: Hello Bosko
```

此输出是来自库和使用库的图表的合并输出。这个概念可以扩展到其他资源，包括那些更长和更复杂的资源。

# 对值文件进行模式化处理

由*values.yaml*文件定义的值是无模式的。并不存在所有*values.yaml*文件需要遵循的固定结构。不同的图表具有不同的结构。这使您可以将值结构化到您使用图表部署的应用程序或工作负载中。

模式提供了许多有用的好处，包括验证内容的能力，您可以执行各种操作，如生成用户界面。

Helm 提供了每个图表可选的能力，使用[JSON Schema](https://json-schema.org)为其值提供自己的模式。JSON Schema 提供了描述 JSON 文件的词汇。YAML 是 JSON 的超集，您可以在这两种文件格式之间转换内容。这使得可以使用 JSON 模式验证 YAML 文件的内容。

当您运行 `helm install`、`helm upgrade`、`helm lint` 和 `helm template` 命令时，Helm 将根据 *values.schema.json* 文件验证值。Helm 验证的值是计算出的值。它们包括图表提供的值以及安装图表的人提供的值。*values.schema.json* 文件与图表根目录下的 *values.yaml* 文件相邻。该文件可以描述所有或部分值。

考虑来自 *values.yaml* 文件的以下部分：

```
image:
  repository: ghcr.io/masterminds/learning-helm/anvil-app
  pullPolicy: IfNotPresent
  tag: ""
```

可以使用以下 JSON Schema 进行检查：

```
{
    "$schema": "http://json-schema.org/schema#",
    "type": "object",
    "properties": {
        "image": {
            "type": "object", ![1](img/1.png)
            "properties": {
                "pullPolicy": {
                    "type": "string", ![2](img/2.png)
                    "enum": ["Always", "IfNotPresent"] ![3](img/3.png)
                },
                "repository": {
                    "type": "string"
                },
                "tag": {
                    "type": "string"
                }
            }
        }
    }
}
```

![1](img/#co_advanced_chart_features_CO4-1)

`image` 是一个对象。如果将 `image` 作为非对象的内容传递给 Helm，将会抛出错误。

![2](img/#co_advanced_chart_features_CO4-2)

`pullPolicy` 是一个字符串。当传递其他类型，比如整数时，将抛出错误。这可以捕捉到一些微妙的问题。

![3](img/#co_advanced_chart_features_CO4-3)

`pullPolicy` 必须是列出的值之一。当传入其他值，甚至拼写错误时，将抛出错误。

为了说明这一点，我们可以使用 *booster* 图表。如果您在图表根目录下运行该命令，您将看到一个错误：

```
$ helm lint . --set image.pullPolicy=foo
```

以下错误告诉您值与模式不匹配的位置：

```
==> Linting .
[ERROR] templates/: values don't meet the specifications of the schema(s) in the
following chart(s):
booster:
- image.pullPolicy: image.pullPolicy must be one of the following: "Always",
  "IfNotPresent"

Error: 1 chart(s) linted, 1 chart(s) failed
```

JSON Schema 提供了几种描述属性的方法。最灵活的方法（一个通用的方法）是为字符串使用正则表达式。例如，可以使用 `^(Always|IfNotPresent)$` 的 `pattern` 来替代 `enum`。这种模式不会像描述性那么强。错误会指出该值不符合模式。在没有其他方法描述属性值时，可以使用模式。

Schema 是图表的有用补充，可以捕捉并修正在安装图表时可能出现的一些微妙问题。

# Hooks

Helm 提供了一种方法来钩入发布过程中的事件并采取行动。如果您想要将动作捆绑为发布的一部分，例如，在升级过程中构建数据库备份的能力，并确保备份在升级 Kubernetes 资源之前发生，这将非常有用。

Hooks 类似于常规模板，它们封装的功能通过运行在 Kubernetes 集群中的容器提供给应用程序的其他资源。钩子与其他资源的区别在于设置了特殊的注解时。当 Helm 看到 `helm.sh/hook` 注解时，它将资源用作钩子，而不是作为图表安装的一部分要安装的资源。表 6-1 包含一些钩子及其执行时机的列表。

表 6-1\. Helm 钩子

| 注解值 | 描述 |
| --- | --- |
| pre-install | 在资源被渲染但上传到 Kubernetes 之前执行。 |
| post-install | 执行发生在将资源上传到 Kubernetes 后。 |
| pre-delete | 执行发生在删除请求之前，任何资源从 Kubernetes 中被删除。 |
| post-delete | 执行发生在从 Kubernetes 中删除所有资源之后。 |
| pre-upgrade | 执行发生在渲染资源之后但在更新 Kubernetes 资源之前。 |
| post-upgrade | 执行发生在 Kubernetes 中的资源升级后。 |
| pre-rollback | 执行发生在渲染资源之后但在 Kubernetes 中任何资源被回滚之前。 |
| post-rollback | 执行发生在 Kubernetes 中的资源被回滚后。 |
| test | 执行发生在运行 `helm test` 命令时。测试将在下一节中讨论。 |

单个资源可以通过将它们列为逗号分隔列表来实现多个钩子。例如：

```
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
```

钩子可以被加权，并在运行后指定资源的删除策略。权重允许为同一事件指定多个钩子，并指定它们运行的顺序。这使您能够确保确定性顺序。因为 Kubernetes 资源用于执行钩子，所以即使执行完成后，这些资源也会存储在 Kubernetes 中。删除策略允许您在何时从 Kubernetes 中删除这些资源时具有一些额外控制。

下面的代码提供了指定所有三个值的示例注释：

```
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
  "helm.sh/hook-weight": "1"
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

使用 `helm.sh/hook-weight` 注释键指定的权重是一个表示为字符串的数字。它应始终为字符串。权重可以是正数或负数，并且默认值为 `0`。在执行钩子之前，Helm 将它们按升序排序。

使用注释键 `helm.sh/hook-delete-policy` 设置的删除策略是一个逗号分隔的策略选项列表。三种可能的删除策略可以在 表 6-2 中找到。

表 6-2\. Helm 钩子删除策略

| 策略值 | 描述 |
| --- | --- |
| before-hook-creation | 在启动新钩子实例之前删除先前的资源。这是默认行为。 |
| hook-succeeded | 在成功运行钩子之后删除 Kubernetes 资源。 |
| hook-failed | 如果在执行钩子时失败，则删除 Kubernetes 资源。 |

默认情况下，Helm 会保留用于钩子的 Kubernetes 资源，直到再次运行钩子。这提供了在运行后检查日志或查看钩子其他信息的能力。设置常见策略的一个例子是前面示例中使用的策略。这将保留钩子资源，除非它们成功完成。当钩子失败时，资源及其日志仍然可供检查，否则它们将被删除。

下面的 `Pod` 是一个运行 post-install 钩子的示例：

```
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-post-install"
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox
      command: ["/bin/sleep","{{ default "10" .Values.sleepTime }}"]
  restartPolicy: Never
```

如果您正在运行 `helm install` 等 Helm 命令，并希望跳过运行钩子，可以使用 `--no-hooks` 标志。此标志适用于具有钩子的命令，并将导致 Helm 跳过执行它们。钩子是一种选择退出的功能。

# 向图表添加测试

测试是软件开发的一个重要部分，Helm 通过 *test* 钩子和 Kubernetes 资源提供了测试图表的能力。这意味着测试在 Kubernetes 集群中与工作负载并行运行，并且可以访问图表安装的组件。除了 Helm 内置的图表测试功能外，Helm 项目还提供了一个名为 Chart Testing 的额外测试工具。由于 Chart Testing 建立在 Helm 客户端的功能之上，我们首先看一下 Helm 客户端内置的功能。

## Helm 测试

Helm 有一个 `helm test` 命令，用于在图表的运行实例上执行测试钩子。实现这些钩子的资源可以检查数据库访问、数据库架构是否正确放置、工作负载之间的工作连接以及其他操作细节。

如果测试失败，Helm 将以非零退出代码退出，并向您提供失败的 Kubernetes 资源的名称。非零退出代码在与某些自动化测试系统配对时非常有用，这些系统通过这种方式检测失败。当您有 Kubernetes 资源的名称时，可以查看日志以查看失败的原因。

测试通常位于 *templates* 目录的 *tests* 子目录中。将测试放置在此目录中提供了有用的分离。这是一种约定，但不是测试运行的必需条件。

为了说明一个测试，我们将看一下[*booster* 图表](https://oreil.ly/COJ7w)。在 *templates/tests* 目录中，有一个名为 *test-connection.yaml* 的单个测试，其中包含以下测试钩子：

```
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "booster.fullname" . }}-test-connection"
  labels:
    {{- include "booster.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "booster.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

此测试是在运行 `helm create` 时为 Nginx 创建的默认测试。它也恰好适用于测试与 booster 应用程序的连接。这个简单的测试说明了测试的结构。

###### 注意

如果您查看一些现有图表中的测试，可能会发现它们使用的钩子是 `test-success` 而不是 `test`。在 Helm 版本 2 中，有一个名为 `test-success` 的钩子用于运行测试。Helm 版本 3 提供了向后兼容性，并将此钩子名称作为测试运行。

运行测试有两个步骤。第一步是安装图表，以便其实例正在运行。您可以使用 `helm install` 命令来执行此操作。以下命令安装 *booster* 图表，并假设您从图表的根目录运行它：

```
$ helm install boost .
```

当图表的实例正在运行时，您可以运行 `helm test` 命令来执行测试：

```
$ helm test boost
```

Helm 在执行期间会输出测试的状态，完成后会提供关于测试和发布的信息。对于之前的测试，它将返回：

```
Pod boost-booster-test-connection pending
Pod boost-booster-test-connection pending
Pod boost-booster-test-connection running
Pod boost-booster-test-connection succeeded
NAME: boost
LAST DEPLOYED: Tue Jul 21 06:47:05 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     boost-booster-test-connection
Last Started:   Tue Jul 21 06:47:12 2020
Last Completed: Tue Jul 21 06:47:17 2020
Phase:          Succeeded
NOTES:
1\. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l
    "app.kubernetes.io/name=booster,app.kubernetes.io/instance=boost"
    -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

当图表有依赖项且这些依赖项有测试时，这些测试也将被运行。例如，如果在本章中使用的 *rocket* 图表的测试被运行，则将运行 *booster* 图表的测试和 *rocket* 图表的测试。

###### 提示

如果您需要在测试的一部分安装配置，则可以将测试挂钩放在 Kubernetes 的 `Secret` 或 ConfigMap 上，以便与其他测试资源一起安装。

测试图表是确保图表内容能够在 Kubernetes 中运行工作负载并捕捉可能破坏该内容的变化的绝佳方式。

## Chart Testing 工具

Helm 项目提供了一个基于 `helm test` 建立的附加测试工具，提供了更高级的测试能力。它包括的一些附加功能有：

+   能够在图表安装时测试不同的、互斥的配置选项。

+   *Chart.yaml* 模式验证包括自定义模式规则。

+   添加了可配置规则的 YAML 检查工具。例如，您可以确保 YAML 文件中的缩进保持一致。

+   当源代码存储在 Git 中时，可以检查 *Chart.yaml* 文件中的 `version` 属性是否已正确递增。

+   能够处理图表集合，并仅测试已更改的图表。

Chart Testing 工具设计用于在持续集成系统工作流中使用，并且其中一些功能直接针对此情况。

Chart Testing 能够测试具有不同、互斥配置的图表，需要知道这些配置。这些配置捆绑在图表的 *ci* 目录中。

在 *ci* 目录中，您可以为每种测试情况创建一个值文件。在命名每个文件时，您需要使用 glob 命名模式 **-values.yaml*。例如，您可以使用类似 *minimal-values.yaml* 和 *full-values.yaml* 的文件名。

Chart Testing 将分别测试每个配置。例如，在进行图表的语法检查时，将分别检查每个案例。使用 `--values` 标志将自定义值传递给 `helm lint`。当运行时测试图表时，相同的想法和标志适用，因为这是最终用户安装图表时提供其自定义配置的方式。

如果您想使用各种配置进行测试，但不希望将这些配置作为图表存档的一部分发布，您可以将 *ci* 目录放入 *.helmignore* 文件中。当 Helm 打包图表时，将忽略 *ci* 目录。

Chart Testing 可以以各种方式安装和使用。例如，您可以将其作为开发系统上的二进制应用程序使用，或者在持续集成系统中的容器中使用。了解更多关于 [如何在您的情况下使用和设置它的信息，请查看项目页面](https://oreil.ly/sJXpR)。

# 安全考虑

一些最大和最值得信赖的技术组织曾经通过软件更新受到用户的攻击。软件更改和用于更新甚至安装软件的机制提供了一个攻击通道。

Helm 提供了一种选择加入的方式来检查图表的来源和完整性。*来源* 提供了验证图表来自公司或个人等的方式，而 *完整性* 则提供了检查您是否收到了未经更改的预期内容的方法。这种功能使您和使用您图表的人可以验证它们的来源，并且内容没有改变。

为了实现这一点，Helm 使用了 Pretty Good Privacy (PGP)、哈希和一个坐落在图表归档文件旁边的来源文件。例如，如果您有一个名为 *mylib-1.0.0.tgz* 的图表归档文件，则可以有一个名为 *mylib-1.0.0.tgz.prov* 的来源文件。该文件包含了一个带有 *Chart.yaml* 文件内容以及图表归档的哈希的 PGP 消息。Helm 可以为您生成这些文件。以下示例是 *mylib-1.0.0.tgz* 的来源文件：

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

apiVersion: v2
description: an example library chart
name: mylib
type: library
version: 0.1.0

...
files:
  mylib-0.1.0.tgz: sha256:d312aea39acf7026f39edebd315a11a52d29a9
                          6a8d68737ead110ac0c0a1163d
-----BEGIN PGP SIGNATURE-----

iQIzBAEBCgAdFiEEcR8o1RDh4Ly9X2v+lDboC/ukaQkFAl8yiesACgkQlDboC/uk
aQkG2BAAlIEgGI7uu9Kr8j4ZIxDseLmgphhPM1kgnIMPriLieBxFXSJQxciN3+dx
OQpIfdsFQvW98EnJ4781Pm+leHY2iI/L08O1cQWUtzKhfPEWC65YQJPXkTKpHnC2
wXYVUVYWvhx6BJ77RiS/f+hoXiC+i1aBqqS0TAG+AqXuwARO2tY/L7cF6EHjsUwD
pPuTNpYZ/OEWqh1KEYZYVDvLm6uN6QjV4pNTFfAgnvMckfoDLQ+kOPQVqCeUWG3F
tZO3sBzUg+Ak2dDviSTOFQ7TCifc3tOOaWS1XtcooSOkUENmTeeWV56jZnhK1rT4
yaIGT16zXZIdmkZ1t5o9VccuAhQ1Us2FhipdGqpD8yDoJABVz/ee9d2zoX8anfR7
LZ7fwecgQ/THnj54RroyQlzf2aottFiL9ZV4MjUqs0CSoA9+SZ/CcJDd/rxBGI8C
yxRqo0VoNdjT8Kr9hha13krfwD8IpLH8bv4kWt3Ckh6rgphjUL19xyTHJY7w2toY
bAeZMl3Y05Ca76EA7XDdoltE57SUS1Zzd+wDRzRD0IZO8KVk+Z5/PzzvV4l9lnDJ
X63fptInbJpyk0xYKLMFquOY7Yy5mlI9de7424CScePo9Nua3GAakfi4zk3i4Auz
2eaoU/S5uXt605OydkSLLz99BAyJwmazzf/qPyYcPWMw/b+gHxw=
=pRcC
-----END PGP SIGNATURE-----
```

来源文件是一个具有特定结构的 PGP 签名消息。消息中的哈希由 Helm 用于验证完整性，PGP 签名用于验证来源。

使用来源文件有两个步骤。首先，您需要生成它们。为此，您需要有一个 PGP 密钥对。

在使用 `helm package` 命令创建包时，可以告诉 Helm 对包进行签名：

```
$ helm package --sign --key 'bugs@acme.example.com' \
  --keyring path/to/keyring mychart
```

附加标志将告诉 Helm 创建来源文件。`--sign` 标志选择签名，`--key` 标志指定要使用的私钥名称，`--keyring` 标志指定要使用的包含用于签名的私钥的密钥环的位置。当 Helm 创建图表的归档时，它还将创建与之并列的 *.prov* 文件。

然后应上传来源文件，与图表归档一起提供下载，放置在图表存储库中。

验证是反向进行的，并内置到命令中，如 `helm install`、`helm upgrade` 和 `helm pull`，以及 `helm verify` 命令中。

Helm 能够处理您本地同时拥有归档和来源文件以及图表位于远程存储库的情况。

为了说明同时拥有这两个文件的情况，我们可以使用 `helm verify` 命令：

```
$ helm verify --keyring path/to/keyring mychart-0.1.0.tgz
```

`verify` 命令将告诉 Helm 检查哈希和签名。`--keyring` 标志告诉 Helm 公钥环的位置，该环中包含与图表签名使用的私钥匹配的公钥。这可以是公钥环或非 ASCII-Armored 版本的公钥。Helm 将查找 *mychart-0.1.0.tgz.prov* 文件，并使用它执行检查。

在 *mylib* 图表上运行 `verify` 命令如下所示：

```
$ helm verify mylib-0.1.0.tgz --keyring public.key
```

这将输出：

```
Signed by: Matthew Farina
Using Key With Fingerprint: 672C657BE06B4B30969C4A57461449C25E36B98E
Chart Hash Verified: sha256:d312aea39acf7026f39edebd315a11a52d29a96a8d68737ead11
                            0ac0c0a1163d
```

如果在 Helm 仓库中有一个图表，Helm 在下载图表时会同时下载验证文件。例如：

```
$ helm install --verify --keyring public.key myrepo/mychart
```

当 Helm 获取图表存档时，还会下载验证文件、验证签名并验证哈希值。

###### 提示

公钥应通过不同的渠道与图表和验证文件共享。

如果在验证过程中出现问题，Helm 将提供错误消息并以非零退出代码退出。

确保图表来自预期之处并且内容未更改，是确保软件供应链安全的有用步骤。

# 自定义资源定义

Kubernetes 自定义资源定义（CRD）提供了扩展 Kubernetes API 的方法，而 Helm 提供了将它们作为图表的一部分安装的方法。

有两种基于 Helm 的方法来管理图表使用的 CRD。选择使用的方法通常取决于需要安装您的图表的人员的要求和环境配置。

首先，*crds*目录是您可以添加到图表中以保存您的 CRD 的特殊目录。Helm 会在安装其他资源之前安装 CRD。这确保了 CRD 在图表中的任何自定义资源或控制器使用之前可用。

*crds*目录中的 CRD 与 Helm 安装的其他资源不同。这些文件没有模板化。这对我们稍后将要涵盖的 CRD 管理工作流程非常有用。Helm 不会像处理其他资源那样升级或删除 CRD。升级 CRD 会更改集群中所有自定义资源实例的 API 表面，而删除 CRD 会移除所有用户的所有自定义资源。在处理这些集群范围的更改时，您需要使用伴随工具，如 Kubernetes 的命令行工具`kubectl`。

因为 CRD 改变了 Kubernetes API，安装您的图表的人可能没有权限安装、升级或删除它们。如果将应用程序捆绑用于分发给其他公司或一般公众，则情况是如此。一些集群管理员通过安全访问控制限制对这些功能的访问。

*crds*目录中的 CRD 可以从图表中提取，并可以直接与`kubectl`等工具一起使用。这样，如果安装图表的人没有权限，可以将提取的 CRD 传递给有权限安装它们的人。提取的 CRD 也可以使用其他工具在集群中升级 CRD。

第二种基于 Helm 的管理 CRD 的方法是使用第二个图表来安装 CRD，以确保在使用自定义资源之前安装 CRD。这种方法通过 Helm 提供了更加精细的控制。

使用第二个图表可以让您：

1.  使用 Helm 模板和正常的*模板*目录来管理 CRD。

1.  Helm 将管理 CRD 的生命周期，包括卸载和升级。如果您希望在卸载图表后保留 CRD 安装状态，可以设置注解 `"helm.sh/resource-policy": keep`，告诉 Helm 跳过卸载资源的步骤。

1.  如果您在应用程序中遇到问题，并使用卸载和重新安装方法尝试修复问题，则独立图表中的 CRD 将不会被删除。

可以通过松耦合或紧耦合安装第二个图表。如果将持有 CRD 的图表设置为依赖项，则使用情况应该是仅安装一次，因为它正在设置集群范围的资源。

在 Helm 管理 CRD 时，需要特别注意处理升级和删除的情况。例如，如果安装了 CRD 的图表的两个版本，则需要确保旧版本不会覆盖新版本，并且新版本不会破坏集群中其他使用旧版本的用户的功能。如果两个不同版本的安装 CRD 的图表由两个不同的人安装，则可能会发生这种情况。在多租户集群中，集群的不同用户可能不知道彼此的存在，因此确保一个集群用户不会破坏另一个集群用户的工作负载非常重要。

在安装和使用 CRD 时，Helm 开发人员建议在整个生命周期的所有步骤中特别小心，以确保图表的用户不会意外地破坏生产工作负载。

# 结论

Helm 图表不仅仅是模板的集合。它们处理依赖关系，可以包含模式，提供事件钩子机制，可以包含测试，并具有安全特性。这些特性使 Helm 成为解决软件包管理问题的强大且可靠的解决方案的一部分。
