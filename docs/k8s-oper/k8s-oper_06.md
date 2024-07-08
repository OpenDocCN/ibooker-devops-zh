# 第六章：适配器运算符

考虑从头开始编写 Operator 可能需要的众多步骤。您将不得不创建 CRDs 来指定最终用户的接口。Kubernetes 控制器不仅需要用 Operator 的领域特定逻辑编写，还需要正确地连接到运行中的集群以接收适当的通知。还需要创建角色和服务账户以允许 Operator 按其需要的方式运行。Operator 作为集群内的一个 pod 运行，因此需要构建一个镜像，以及其伴随的部署清单。

许多项目已经投资于应用程序部署和配置技术。Helm 项目允许用户在格式化的文本文件中定义其集群资源，并通过 Helm 命令行工具部署它们。Ansible 是一个流行的自动化引擎，用于创建可重用脚本来配置和配置一组资源。这两个项目都有专注的开发者群体，他们可能缺乏资源来迁移到使用 Operator 来管理其应用程序。

Operator SDK 通过其*适配器运算符*解决了这两个问题。通过命令行工具，SDK 生成运行 Helm 和 Ansible 等技术所需的代码。这使您能够快速将基础架构迁移到 Operator 模型，无需编写必要的支持 Operator 代码。这样做的优势包括：

+   通过 CRDs 提供一致的接口，无论底层技术是 Helm、Ansible 还是 Go。

+   允许 Helm 和 Ansible 解决方案利用 Operator 生命周期管理器提供的部署和生命周期优势（有关更多信息，请参见第八章）。

+   允许将这些解决方案托管在 OperatorHub.io 等 Operator 仓库上（有关更多信息，请参见第十章）。

在本章中，我们展示了如何使用 SDK 在前一章介绍的访客网站应用程序中构建和测试适配器运算符。

# Helm 运算符

[Helm](https://helm.sh/)是 Kubernetes 的一个包管理工具。它简化了部署具有多个组件的应用程序，但每次部署仍然是一个手动过程。如果您正在部署多个 Helm 打包的应用程序实例，使用 Operator 自动化这些部署将非常方便。

Helm 的复杂性超出了本书的范围——您可以查阅[文档](https://helm.sh/docs/)以获取更多详细信息——但一些背景知识将帮助您理解 Helm Operator。Helm 定义了构成应用程序的 Kubernetes 资源，如部署和服务，在一个称为*chart*的文件中。图表支持配置变量，因此您可以自定义应用程序实例而无需编辑图表本身。这些配置值在名为*values.yaml*的文件中指定。Helm Operator 可以使用不同版本的*values.yaml*部署每个应用程序实例。

当传递`--type=helm`参数时，Operator SDK 为 Helm Operator 生成 Kubernetes 控制器代码。您为应用程序提供一个 Helm 图表，生成的 Helm Operator 会监视其给定类型的新 CR。当它找到其中一个 CR 时，它会根据资源中定义的值构建一个 Helm *values.yaml*文件。然后，Operator 根据*values.yaml*中的设置创建其 Helm 图表中指定的应用程序资源。要配置另一个应用程序实例，您需要创建一个包含适当值的新 CR。

SDK 提供了两种构建基于 Helm 的 Operator 的变体：

+   项目生成过程在 Operator 项目代码中构建了一个空白的 Helm 图表结构。

+   在 Operator 创建时指定现有图表，创建过程将使用它来填充生成的 Operator。

在接下来的章节中，我们将讨论这些方法的每一个。作为先决条件，请确保在您的计算机上安装了 Helm 命令行工具。您可以在[Helm 的安装文档](https://oreil.ly/qpZX0)中找到有关如何执行此操作的信息。

## 构建 Operator

SDK 的`new`命令为新 Operator 创建了框架项目文件。这些文件包括调用适当的 Helm 图表以处理 CR 请求的 Kubernetes 控制器所需的所有代码。我们稍后将在本节更详细地讨论这些文件。

### 创建一个新图表

使用`--type=helm`参数创建一个具有新 Helm 图表框架的 Operator。以下示例创建了一个访客站点应用程序的 Helm Operator 的基础（参见第五章）：

```
$ `OPERATOR_NAME``=``visitors-helm-operator`
$ `operator-sdk` `new` `$OPERATOR_NAME` `--api-version``=``example.com/v1` `\ `  `--kind``=``VisitorsApp` `--type``=``helm`
INFO[0000] Creating new Helm operator 'visitors-helm-operator'.
INFO[0000] Created helm-charts/visitorsapp
INFO[0000] Generating RBAC rules
WARN[0000] The RBAC rules generated in deploy/role.yaml are based on
the chart's default manifest. Some rules may be missing for resources
that are only enabled with custom values, and some existing rules may
be overly broad. Double check the rules generated in deploy/role.yaml
to ensure they meet the operator's permission requirements.
INFO[0000] Created build/Dockerfile
INFO[0000] Created watches.yaml
INFO[0000] Created deploy/service_account.yaml
INFO[0000] Created deploy/role.yaml
INFO[0000] Created deploy/role_binding.yaml
INFO[0000] Created deploy/operator.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_crd.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_cr.yaml
INFO[0000] Project creation complete.

```

`visitors-helm-operator`是生成的 Operator 的名称。另外两个参数，`--api-version`和`--kind`，描述了此 Operator 管理的 CR。这些参数导致为新类型创建基本 CRD。

SDK 创建一个与`$OPERATOR_NAME`相同名称的新目录，其中包含所有 Operator 的文件。有几个文件和目录需要注意：

*deploy/*

此目录包含用于部署和配置 Operator 的 Kubernetes 模板文件，包括 CRD、Operator 部署资源本身以及 Operator 运行所需的 RBAC 资源。

*helm-charts/*

此目录包含与 CR 类型同名的 Helm chart 的骨架目录结构。其中的文件与 Helm CLI 在初始化新图表时创建的文件类似，包括一个*values.yaml*文件。每个 Operator 管理的新 CR 类型都将向此目录添加一个新图表。

*watches.yaml*

该文件将每种 CR 类型映射到用于处理它的特定 Helm chart。

现在，一切都准备就绪，可以开始实现您的图表。但是，如果您已经编写了一个图表，还有一种更简单的方法。

### 使用现有图表

从现有 Helm chart 构建 Operator 的过程与使用新 chart 创建 Operator 的过程类似。除了`--type=helm`参数外，还有一些额外的参数需要考虑：

--helm-chart

告知 SDK 使用现有图表初始化 Operator。该值可以是：

+   图表存档的 URL

+   远程图表的存储库和名称

+   本地目录的位置

--helm-chart-repo

指定图表的远程存储库 URL（除非另有指定本地目录）。

--helm-chart-version

告知 SDK 获取特定版本的图表。如果省略此项，则使用最新可用版本。

在使用`--helm-chart`参数时，`--api-version`和`--kind`参数变为可选项。`api-version`默认为`charts.helm.k8s.io/v1alpha1`，并且`kind`名称将根据 chart 名称推断而来。然而，由于`api-version`携带有关 CR 创建者的信息，我们建议您明确地适当填充这些值。您可以在本书的[Github 存储库](https://github.com/kubernetes-operators-book/chapters/tree/master/ch06/visitors-helm)中找到部署 Visitors Site 应用程序的示例 Helm chart。

下面的示例演示了如何构建一个 Operator，并使用 Visitors Site Helm chart 的存档进行初始化：

```
$ `OPERATOR_NAME``=``visitors-helm-operator`
$ `wget` `https://github.com/kubernetes-operators-book/``\ `  `chapters/releases/download/1.0.0/visitors-helm.tgz`  ![1](img/1.png)
$ `operator-sdk` `new` `$OPERATOR_NAME` `--api-version``=``example.com/v1` `\ `  `--kind``=``VisitorsApp` `--type``=``helm` `--helm-chart``=``./visitors-helm.tgz`
INFO[0000] Creating new Helm operator 'visitors-helm-operator'.
INFO[0000] Created helm-charts/visitors-helm
INFO[0000] Generating RBAC rules
WARN[0000] The RBAC rules generated in deploy/role.yaml are based on
the chart's default manifest. Some rules may be missing for resources
that are only enabled with custom values, and some existing rules may
be overly broad. Double check the rules generated in deploy/role.yaml
to ensure they meet the operator's permission requirements.
INFO[0000] Created build/Dockerfile
INFO[0000] Created watches.yaml
INFO[0000] Created deploy/service_account.yaml
INFO[0000] Created deploy/role.yaml
INFO[0000] Created deploy/role_binding.yaml
INFO[0000] Created deploy/operator.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_crd.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_cr.yaml
INFO[0000] Project creation complete.

```

![1](img/#comarker1-01a)

由于 Operator SDK 处理重定向的方式存在问题，您必须手动下载图表 tarball 并将其作为本地引用传递。

前述示例生成了与使用新 Helm chart 创建 Operator 时相同的文件，唯一的例外是图表文件是从指定的存档中填充的：

```
$ `ls` `-l` `$OPERATOR_NAME``/helm-charts/visitors-helm/templates`
_helpers.tpl
auth.yaml
backend-deployment.yaml
backend-service.yaml
frontend-deployment.yaml
frontend-service.yaml
mysql-deployment.yaml
mysql-service.yaml
tests

```

SDK 使用图表的*values.yaml*文件中的值填充示例 CR 模板。例如，Visitors Site Helm chart 具有以下*values.yaml*文件：

```
$ `cat` `$OPERATOR_NAME``/helm-charts/visitors-helm/values.yaml`
backend:
  size: 1

frontend:
  title: Helm Installed Visitors Site

```

SDK 生成的示例 CR 位于 Operator 项目根目录中的*deploy/crds*目录中，并在其`spec`部分包含相同的值：

```
$ `cat` `$OPERATOR_NAME``/deploy/crds/example_v1_visitorsapp_cr.yaml`
apiVersion: example.com/v1
kind: VisitorsApp
metadata:
  name: example-visitorsapp
spec:
  # Default values copied from <proj_dir>/helm-charts/visitors-helm/values.yaml 
  backend:
    size: 1

  frontend:
    title: Helm Installed Visitors Site

```

在运行图表之前，Operator 将将自定义资源的`spec`字段中的值映射到*values.yaml*文件中。

## Fleshing Out the CRD

生成的 CRD 不包括 CR 类型的值输入和状态值的具体细节。附录 B 描述了完成 CR 定义的步骤。

## 审查操作员权限

生成的部署文件包括操作员将用于连接 Kubernetes API 的角色。默认情况下，此角色权限非常宽泛。附录 C 讨论如何精细调整角色定义以限制操作员的权限。

## 运行 Helm Operator

操作员作为普通容器映像交付。然而，在开发和测试周期中，跳过映像创建过程并在集群外部简单运行操作员通常更容易。本节描述了这些步骤（请参阅 附录 A 了解有关在集群内部署操作员的信息）。请从操作员项目根目录内运行所有命令：

1.  创建本地观察文件。生成的 *watches.yaml* 文件引用特定路径，该路径在部署的操作员场景中与 Helm chart 相对应。镜像创建过程负责将 chart 复制到必要的位置。当在集群外运行操作员时，仍需要此 *watches.yaml* 文件，因此您需要手动确保 chart 可以在该位置找到。

    最简单的方法是复制已存在于操作员项目根目录中的 *watches.yaml* 文件：

    ```
    $ `cp` `watches.yaml` `local``-watches.yaml`

    ```

    在 *local-watches.yaml* 文件中，编辑 `chart` 字段，以包含您机器上 Helm chart 的完整路径。请记住本地观察文件的名称；稍后在启动操作员过程中会用到它。

1.  使用 `kubectl` 命令在集群中创建 CRDs：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_crd.yaml`

    ```

1.  创建完 CRDs 后，使用以下 SDK 命令启动操作员：

    ```
    $ `operator-sdk` `up` `local` `--watches-file` `./local-watches.yaml`
    INFO[0000] Running the operator locally.
    INFO[0000] Using namespace default.  ![1](img/1.png)

    ```

    ![1](img/#comarker1-02a)

    当操作员启动并处理 CR 请求时，操作员日志消息将显示在此运行过程中。

    此命令启动一个运行过程，其行为方式与在集群内部署为 pod 的操作员相同。我们将在 “测试操作员” 中更详细地讨论测试。

# Ansible Operator

[Ansible](https://www.ansible.com/) 是一种流行的管理工具，用于自动化常见任务的配置和部署。与 Helm chart 类似，Ansible *playbook* 定义了一系列在一组服务器上运行的 *tasks*。可重用的 *roles* 可通过自定义功能扩展 Ansible，以增强 playbook 中的任务集合。

一个有用的角色集合是 [k8s](https://oreil.ly/1ckgw)，它提供了与 Kubernetes 集群交互的任务。使用此模块，您可以编写 playbook 来处理应用程序的部署，包括所有必要的支持 Kubernetes 资源。

Operator SDK 提供了一种构建 Operator 的方式，该 Operator 将运行 Ansible playbook 以响应 CR 更改。SDK 提供了用于 Kubernetes 组件（例如控制器）的代码，使您可以专注于编写 playbook 本身。

## 构建 Operator

与其 Helm 支持一样，Operator SDK 生成项目框架。使用`--type=ansible`参数运行时，项目框架包含空白 Ansible 角色的结构。角色名称由指定的 CR 类型名称派生。

以下示例演示了创建定义 Visitors Site 应用程序 CR 的 Ansible Operator：

```
$ `OPERATOR_NAME``=``visitors-ansible-operator`
$ `operator-sdk` `new` `$OPERATOR_NAME` `--api-version``=``example.com/v1` `\ `  `--kind``=``VisitorsApp` `--type``=``ansible`
INFO[0000] Creating new Ansible operator 'visitors-ansible-operator'.
INFO[0000] Created deploy/service_account.yaml
INFO[0000] Created deploy/role.yaml
INFO[0000] Created deploy/role_binding.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_crd.yaml
INFO[0000] Created deploy/crds/example_v1_visitorsapp_cr.yaml
INFO[0000] Created build/Dockerfile
INFO[0000] Created roles/visitorsapp/README.md
INFO[0000] Created roles/visitorsapp/meta/main.yml
INFO[0000] Created roles/visitorsapp/files/.placeholder
INFO[0000] Created roles/visitorsapp/templates/.placeholder
INFO[0000] Created roles/visitorsapp/vars/main.yml
INFO[0000] Created molecule/test-local/playbook.yml
INFO[0000] Created roles/visitorsapp/defaults/main.yml
INFO[0000] Created roles/visitorsapp/tasks/main.yml
INFO[0000] Created molecule/default/molecule.yml
INFO[0000] Created build/test-framework/Dockerfile
INFO[0000] Created molecule/test-cluster/molecule.yml
INFO[0000] Created molecule/default/prepare.yml
INFO[0000] Created molecule/default/playbook.yml
INFO[0000] Created build/test-framework/ansible-test.sh
INFO[0000] Created molecule/default/asserts.yml
INFO[0000] Created molecule/test-cluster/playbook.yml
INFO[0000] Created roles/visitorsapp/handlers/main.yml
INFO[0000] Created watches.yaml
INFO[0000] Created deploy/operator.yaml
INFO[0000] Created .travis.yml
INFO[0000] Created molecule/test-local/molecule.yml
INFO[0000] Created molecule/test-local/prepare.yml
INFO[0000] Project creation complete.

```

此命令生成类似于 Helm Operator 示例的目录结构。SDK 创建包含 CRD 和部署模板等相同文件集的*deploy*目录。

与 Helm Operator 相比，有几个显著的区别：

*watches.yaml*

+   这样做的目的与 Helm Operator 相同：它将 CR 类型映射到在其解析期间执行的文件位置。但是，Ansible Operator 支持两种不同类型的文件（这些字段是互斥的）：

    +   如果包含`role`字段，则必须指向在资源协调期间执行的 Ansible *role*目录。

    +   如果包含`playbook`字段，则必须指向要运行的*playbook*文件。

+   SDK 默认将此文件指向在生成期间创建的角色。

*roles/*

+   此目录包含所有操作员可能运行的 Ansible 角色。项目创建时，SDK 会生成新角色的基本文件。

+   如果 Operator 管理多个 CR 类型，则将多个角色添加到此目录中。此外，对于每种类型及其关联的角色，watches 文件都会添加一个条目。

接下来，您将为 CR 实现 Ansible 角色。角色的具体操作取决于应用程序：一些常见任务包括创建部署和服务以运行应用程序的容器。有关编写 Ansible 角色的更多信息，请参阅[Ansible 文档](https://oreil.ly/bLd5g)。

您可以在本书的[GitHub 存储库](https://github.com/kubernetes-operators-book/chapters/tree/master/ch06/ansible/visitors)中找到部署 Visitors Site 的 Ansible 角色。为了简便起见，按照示例应用程序进行跟随时，角色文件作为发布可用。与以前的 Operator 创建命令类似，您可以通过以下方式添加 Visitors Site 角色：

```
$ `cd` `$OPERATOR_NAME``/roles/visitorsapp`
$ `wget` `https://github.com/kubernetes-operators-book/``\ `  `chapters/releases/download/1.0.0/visitors-ansible.tgz`
$ `tar` `-zxvf` `visitors-ansible.tgz` ![1](img/1.png)
$ `rm` `visitors-ansible.tgz`

```

![1](img/#comarker1-03a)

此命令使用运行 Visitors Site 角色所需的文件覆盖默认生成的角色文件。

本书不涵盖编写 Ansible 角色，但重要的是您了解如何将用户输入的配置值传播到 Ansible 角色中。

与 Helm 操作员类似，配置值来自 CR 的 `spec` 部分。然而，在 Playbooks 和 Roles 中，Ansible 使用标准的 `{{ `variable_name` }}` 语法。Kubernetes 中的字段名称通常使用驼峰命名（例如，camelCase），因此 Ansible 操作员会在将参数传递给 Ansible 角色之前将每个字段的名称转换为蛇形命名（例如，snake_case）。也就是说，字段名称 `serviceAccount` 将转换为 `service_account`。这样可以使用标准的 Ansible 约定重用现有的角色，同时也遵循 Kubernetes 资源约定。你可以在书籍的[GitHub 仓库](https://github.com/kubernetes-operators-book/chapters/tree/master/ch06/ansible)中找到部署访问者站点的 Ansible 角色的源代码。

## 扩展自定义资源定义（CRD）

与 Helm 操作员类似，您需要扩展生成的 CRD 来包含您的 CR 的具体信息。有关更多信息，请参阅附录 B。

## 检查操作员权限

Ansible 操作员还包括一个生成的角色，操作员用它来连接 Kubernetes API。更多关于优化默认权限的信息，请查看附录 C。

## 运行 Ansible 操作员

与 Helm 操作员类似，测试和调试 Ansible 操作员的最简单方法是在集群外运行它，避免构建和推送镜像的步骤。

然而，在你执行此操作之前，还有一些额外的步骤你需要完成：

1.  首先，在运行操作员的机器上安装 Ansible。请查阅[Ansible 文档](https://oreil.ly/9yZRC)获取有关如何在您的本地操作系统上安装 Ansible 的具体信息。

1.  还必须安装其他与 Ansible 相关的软件包，包括以下内容（有关安装详细信息，请查阅文档）：

    +   [Ansible Runner](https://oreil.ly/lHDCe)

    +   [Ansible Runner HTTP 事件发射器](https://oreil.ly/N6ebi)

1.  与 Helm 操作员类似，SDK 生成的 *watches.yaml* 文件引用特定目录中的 Ansible 角色。因此，您需要复制 watches 文件并根据需要进行修改。再次强调，从操作员项目根目录中运行这些命令：

    ```
    $ `cp` `watches.yaml` `local``-watches.yaml`

    ```

    在 *local-watches.yaml* 文件中，更改 `role` 字段以反映您计算机上的目录结构。

1.  使用 `kubectl` 命令在集群中创建自定义资源定义（CRD）：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_crd.yaml`

    ```

1.  一旦 CRD 在集群中部署，就可以使用 SDK 运行操作员：

    ```
    $ `operator-sdk` `up` `local` `--watches-file` `./local-watches.yaml`
    INFO[0000] Running the operator locally.
    INFO[0000] Using namespace default.  ![1](img/1.png)

    ```

    ![1](img/#comarker1-04a)

    操作员日志消息将在这个运行中的进程中显示，随着其启动和处理 CR 请求。

    该命令启动一个运行中的进程，其行为类似于如果操作员作为一个 Pod 部署在集群内部的话。

现在让我们一起逐步走过测试操作员的步骤。

# 测试操作员

您可以使用相同的方法测试适配器运算符：通过部署 CR。Kubernetes 会通知运算符 CR 的存在，然后执行底层文件（Helm 图表或 Ansible 角色）。SDK 在 *deploy/crds* 目录中生成一个示例 CR 模板，您可以使用它，或者手动创建一个。

按照以下步骤测试本章讨论的两种运算符类型：

1.  编辑示例 CR 模板的 `spec` 部分（在访问者网站示例中，它命名为 *example_v1_visitorsapp_cr.yaml*），填入与您的 CR 相关的任何值。

1.  使用 Kubernetes CLI 在运算符项目根目录中创建资源：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_cr.yaml`

    ```

    运算符的输出将显示在您运行 `operator-sdk up local` 命令的同一终端中。测试完成后，通过按下 `Ctrl-C` 结束运行进程。

1.  按照 第五章 中描述的步骤导航到访问者网站，以验证应用程序按预期工作。

1.  测试完成后，请使用 `kubectl delete` 命令删除 CR：

    ```
    $ `kubectl` `delete` `-f` `deploy/crds/*_cr.yaml`

    ```

在开发过程中，重复此过程以测试更改。在每次迭代中，请确保重新启动运算符进程，以获取对 Helm 或 Ansible 文件的任何更改。

# 摘要

你无需成为程序员就能编写一个运算符。Operator SDK 简化了将两种现有的配置和配置技术（Helm 和 Ansible）打包为运算符的过程。SDK 还提供了一种快速测试和调试更改的方式，通过在集群之外运行运算符，跳过耗时的镜像构建和托管步骤。

在下一章中，我们将介绍一种更强大和灵活的运算符实现方式，使用 Go 语言。

# 资源

+   [Helm](https://helm.sh/)

+   [Ansible](https://www.ansible.com/)

+   [示例运算符](https://oreil.ly/KbPFs)
