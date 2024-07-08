# 附录 B. 自定义资源验证

在添加新的 API 时，Operator SDK 会生成一个骨架自定义资源定义。这个骨架是可用的；创建自定义资源无需进行进一步的更改或添加。

通过简单地定义 `spec` 和 `status` 部分，分别表示用户输入和自定义资源状态的开放对象，骨架 CRD 实现了这种灵活性：

```
spec:
    type: object
status:
    type: object
```

这种方法的缺点是 Kubernetes 无法验证这两个字段中的任何数据。由于 Kubernetes 不知道应该允许什么值或不允许什么值，只要清单解析，这些值就是允许的。

为解决这个问题，CRD 包括对 [OpenAPI 规范](https://oreil.ly/bzRIu) 的支持，以描述每个字段的验证约束。您需要手动将此验证添加到 CRD 中，以描述 `spec` 和 `status` 部分的允许值。

您将对 CRD 的 `spec` 部分进行两个主要更改：

+   添加一个 `properties` 映射。对于可以为此类型的自定义资源指定的每个属性，将一个条目添加到此映射中，并提供有关参数类型和允许值的信息。

+   可选地，您可以添加一个 `required` 字段，列出 Kubernetes 应强制执行其存在的属性。将每个必需属性的名称作为此列表中的条目添加。如果在资源创建过程中省略了这些属性中的任何一个，Kubernetes 将拒绝该资源。

您还可以按照与 `spec` 相同的约定填充 `status` 部分的属性信息；但是，无需添加 `required` 字段。

###### 警告

在这两种情况下，现有的行 `type: object` 保持不变；您将新添加的内容插入到与此“type”声明相同级别的位置。

您可以在 CRD 的以下部分找到 `spec` 和 `status` 字段：

```
spec -> validation -> openAPIV3Schema -> properties
```

例如，对 VisitorsApp CRD 的添加如下：

```
spec:
    type: object
    properties:
        size:
            type: integer
        title:
            type: string
    required:
    - size
status:
    type: object
    properties:
        backendImage:
            type: string
        frontendImage:
            type: string
```

这个片段只是使用 OpenAPI 验证可以实现的示例。您可以在 [Kubernetes 文档](https://oreil.ly/FfkJe) 中找到有关创建自定义资源定义的详细信息。
