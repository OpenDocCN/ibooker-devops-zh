# 附录 A. 在集群内作为部署运行操作员

在集群外部运行操作员，对于测试和调试目的非常方便，但是生产环境中的操作员以 Kubernetes 部署方式运行。对于此部署方式，涉及一些额外的步骤：

1.  *构建图像*。操作员 SDK 的 `build` 命令链接到底层的 Docker 守护进程以构建操作员图像，并在运行时提供完整的图像名称和版本：

    ```
    $ `operator-sdk` `build` `jdob/visitors-operator:0.1`

    ```

1.  *配置部署*。更新 SDK 生成的 *deploy/operator.yaml* 文件，其中包含图像的名称。要更新的字段名为 `image`，可以在以下位置找到：

    ```
    spec -> template -> spec -> containers
    ```

    生成的文件默认将值设置为 `REPLACE_IMAGE`，您应该更新以反映在上一条命令中构建的图像名称。

    构建完成后，请将图像推送到诸如 [Quay.io](https://quay.io) 或 [Docker Hub](https://hub.docker.com) 这样的外部可访问的仓库。

1.  *部署 CRD*。SDK 生成一个基本的 CRD 框架，可以正常运行，但请参阅 附录 B 获取有关完善此文件的更多信息：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_crd.yaml`

    ```

1.  *部署服务账户和角色*。SDK 生成操作员所需的服务账户和角色。更新这些内容以限制角色的权限，使其仅限于操作员所需的最小权限。

    一旦适当地确定了角色权限的范围，将资源部署到集群中：

    ```
    $ `kubectl` `apply` `-f` `deploy/service_account.yaml`
    $ `kubectl` `apply` `-f` `deploy/role.yaml`
    $ `kubectl` `apply` `-f` `deploy/role_binding.yaml`

    ```

    ###### 警告

    您必须按照列出的顺序部署这些文件，因为角色绑定需要角色和服务账户同时存在。

1.  *创建操作员部署*。最后一步是部署操作员本身。您可以使用先前编辑的 *operator.yaml* 文件将操作员图像部署到集群中：

    ```
    $ `kubectl` `apply` `-f` `deploy/operator.yaml`

    ```
