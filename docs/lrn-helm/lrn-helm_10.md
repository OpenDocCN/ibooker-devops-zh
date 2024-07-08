# 附录 B. 图表仓库 API

在 第七章 中，我们涵盖了图表仓库。本附录简要介绍了图表仓库 API，这是使 Helm 能够与图表仓库配合工作的基础规范。

图表仓库 API 很轻量，因为只需要实现一个必需的 HTTP 端点：`GET /index.yaml`。

在 99% 的情况下，图表仓库还会提供图表包 tarballs (*.tgz*) 和任何相关的确证文件 (*.prov*)。但是，也可以将这些文件托管在不同的域上。

如 第七章 中详细描述的那样，*index.yaml* 表示仓库索引，包含仓库中所有可用图表版本的完整列表。此文件的格式特定于 Helm，并且目前仅支持 API 版本 `1`。

# index.yaml

在实现图表仓库 API 时，您的服务必须提供相对于提供的仓库 URL 的 HTTP `GET /index.yaml` 路由。该请求的响应必须返回状态码 `200 OK`，响应正文必须是一个有效的 *index.yaml*，如下所述。

###### 注意

`GET /index.yaml` 端点不需要位于 URL 路径的根目录下。例如，给定提供的仓库 URL，如 *[*https://example.com/charts*](https://example.com/charts)*，`GET /index.yaml` 路由必须在 *[*https://example.com/charts/index.yaml*](https://example.com/charts/index.yaml)* 可访问。

## index.yaml 格式

以下是一个简单且有效的 *index.yaml*，仅包含一个图表版本（`superapp-0.1.0`）：

```
apiVersion: v1 ![1](img/1.png)
entries: ![2](img/2.png)
  superapp:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2020-04-28T10:12:22.507943-05:00" ![3](img/3.png)
    description: A Helm chart for Kubernetes
    digest: 46f9ddeca12ec0bc257a702dac7d069af018aed2a87314d86b230454ac033672 ![4](img/4.png)
    name: superapp
    type: application
    urls: ![5](img/5.png)
    - superapp-0.1.0.tgz
    version: 0.1.0
generated: "2020-04-28T11:34:26.779758-05:00" ![6](img/6.png)
```

![1](img/#co_chart_repository_api_CO1-1)

仓库 API 版本（必须始终为 `v1`）。

![2](img/#co_chart_repository_api_CO1-2)

一个映射，将仓库中唯一的图表名称映射到所有可用版本的列表。

![3](img/#co_chart_repository_api_CO1-3)

使用 `helm package` 创建 tarball 的时间戳。

![4](img/#co_chart_repository_api_CO1-4)

tarball 的 SHA-256 摘要。

![5](img/#co_chart_repository_api_CO1-5)

可以下载图表的 URL 列表。这些 URL 可以是绝对的，甚至可以托管在不同的域上。如果提供相对路径，则视为相对于 *index.yaml*。通常每个图表版本仅提供一个 URL 条目，但可以提供多个，如果前一个不可访问，Helm 将尝试下载列表中的下一项。

![6](img/#co_chart_repository_api_CO1-6)

生成此 *index.yaml* 文件的时间戳，以 RFC 3339 格式。

除了 `created`、`digest` 和 `urls` 字段外，每个单独的图表版本上的所有字段均由图表 API（`name`、`version` 等）定义。请参阅 附录 A 获取更多信息。

## 何时下载 index.yaml？

当 Helm 下载或重新下载仓库索引时，有五种值得注意的情况：

1.  当最初添加图表仓库时：

    ```
    $ helm repo add myrepo https://charts.example.com
    ```

1.  更新所有图表仓库时：

    ```
    $ helm repo update
    ```

1.  更新依赖项时（使用 `--skip-refresh` 标志禁用）：

    ```
    $ helm dependency update
    ```

1.  从锁文件构建依赖项时（使用 `--skip-refresh` 标志禁用）：

    ```
    $ helm dependency build
    ```

1.  使用 `--dependency-update` 标志安装具有依赖项的本地图表时：

    ```
    $ helm install myapp . --dependency-update
    ```

## 何时使用缓存版本的 index.yaml？

下载 *index.yaml* 后，它将存储在本地缓存中，并在引用与仓库关联的唯一名称时使用（例如，“myrepo”）。

当 Helm 利用本地缓存的仓库索引时，有五种值得注意的情况：

1.  从仓库拉取图表时：

    ```
    helm pull myrepo/mychart
    ```

1.  从仓库安装图表时：

    ```
    helm install myapp myrepo/mychart
    ```

1.  根据来自仓库的图表升级发布时：

    ```
    helm upgrade myapp myrepo/mychart
    ```

1.  当搜索要使用的图表时：

    ```
    helm search repo myrepo/
    ```

1.  使用 `--skip-refresh` 标志更新依赖项（如果依赖项包含 `"@myrepo"` 等 `alias` 子字段时）：

    ```
    helm dependency update --skip-refresh
    ```

# .tgz 文件

仓库中的 *.tgz* 文件代表以压缩 tarball 形式打包的单个图表版本。

对于这些文件，没有 URL 路径的要求，因为它们是托管在仓库中的；但是，当 Helm 请求时，它们必须能够被下载。响应的状态码必须是 `200 OK`，响应体应为二进制形式的 *.tgz* 内容。

## 何时下载 .tgz 文件？

当 Helm 下载图表包 *.tgz* 文件时，有三种值得注意的情况：

1.  从仓库拉取图表时：

    ```
    helm pull myrepo/mychart
    ```

1.  从仓库安装图表时：

    ```
    helm install myapp myrepo/mychart
    ```

1.  根据来自仓库的图表升级发布时：

    ```
    helm upgrade myapp myrepo/mychart
    ```

# .prov 文件

仓库中的 *.prov* 文件代表使用 GNU Privacy Guard 签名的图表版本签名文件，这些文件是 *可选的*，用于验证目的。

与 *.tgz* 文件不同，*.prov* 文件具有唯一的 URL 路径要求。它们必须在与关联的 *.tgz* 后缀 *.prov* 文件位于的路径可访问的位置。例如，如果一个 *.tgz* 文件位于 *https://charts.example.com/superapp-0.1.0.tgz*，则 *.prov* 文件必须位于 *https://charts.example.com/superapp-0.1.0.tgz.prov*。

响应的状态码必须是 `200 OK`，响应体应为二进制形式的 *.prov* 内容。

## 何时下载 .prov 文件？

当 Helm 下载图表签名 *.prov* 文件时，有三种值得注意的情况：

1.  使用 `--verify` 标志从仓库拉取图表时：

    ```
    helm pull myrepo/mychart --verify
    ```

1.  使用 `--verify` 标志从仓库安装图表时：

    ```
    helm install myapp myrepo/mychart --verify
    ```

1.  根据来自带有 `--verify` 标志的仓库图表升级发布时：

    ```
    helm upgrade myapp myrepo/mychart --verify
    ```
