# 第十三章：API 弃用

Kubernetes 项目定期发布新版本。每个发布版本都会增加新功能和错误修复，但也可能引入现有 API 的弃用。API 是应用程序开发人员在定义 Kubernetes 对象时与之交互的接口。

如果 Kubernetes 团队计划更改、替换或完全移除支持的 API，弃用可能会生效。您需要了解如何处理 API 弃用，以避免在将节点更新到较新的 Kubernetes 版本之前出现问题。

# 理解弃用策略

Kubernetes 项目每年会发布 [三个版本](https://kubernetes.io/blog/2021/07/20/new-kubernetes-release-cadence/)。理想情况下，Kubernetes 集群的管理员应尽早升级到最新版本，以便获得增强功能和安全修复。然而，升级集群并非没有潜在的成本和风险。您需要确保在升级到新版本时，现有的运行中对象仍然与之兼容。

Kubernetes 发布版本可能会弃用一个 API，这意味着该 API 计划要被移除或替换。弃用引入的规则遵循 Kubernetes 文档中解释的 [弃用策略](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)。

在清单中为 `version` 属性分配的值指定了 API 版本。使用已弃用的 API 在创建或更新对象时会显示警告消息。虽然您仍然可以创建或修改对象，但警告消息会告知用户采取措施，以确保其与较新的 Kubernetes 版本未来兼容。[已弃用 API 迁移指南](https://kubernetes.io/docs/reference/using-api/deprecation-guide/) 显示了一些已弃用的 API 和计划中将移除其支持的版本列表。

# 列出可用的 API 版本

`Kubectl` 提供了一个命令用于发现可用的 API 版本。`api-versions` 命令以 group/version 的格式列出所有 API 版本：

```
$ kubectl api-versions
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2
batch/v1
certificates.k8s.io/v1
coordination.k8s.io/v1
discovery.k8s.io/v1
events.k8s.io/v1
flowcontrol.apiserver.k8s.io/v1beta2
flowcontrol.apiserver.k8s.io/v1beta3
networking.k8s.io/v1
node.k8s.io/v1
policy/v1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
v1

```

某些 API，如 `autoscaling` 组的 `v1` 和 `v2` 版本，存在不同的版本。一般来说，通过选择更高的主版本可以使您的清单更加未来兼容。在撰写本文时，`api-versions` 命令的输出中不包含这些 API 的弃用状态。您需要在 Kubernetes 文档中查找其状态。

# 处理弃用警告

让我们演示使用已弃用 API 的效果。以下示例假设您正在运行版本在 1.23 到 1.25 之间的 Kubernetes 集群。示例 13-1 展示了一个使用 `autoscaling` 组的 beta API 版本的 Horizontal Pod Autoscaler 的清单。

##### 示例 13-1\. 使用已弃用 API 定义的 Horizontal Pod Autoscaler

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
```

从 YAML 清单创建新对象不会导致错误。Kubernetes 将愉快地创建对象，但命令会告知您在将来的 Kubernetes 版本中该 API 的用法。如下输出所示，建议您替换使用 `autoscaling/v2` 的 API：

```
$ kubectl create -f hpa.yaml
Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, \
unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler

```

除了警告消息外，您可能还希望查看弃用 API 迁移指南。查找关于弃用 API 的信息的简便方法是在弃用指南中搜索它。例如，您将找到关于 API `autoscaling/v2beta2` 的 [以下段落](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#horizontalpodautoscaler-v126)：

> HorizontalPodAutoscaler 的 `autoscaling/v2beta2` API 版本已在 v1.26 中停止服务。
> 
> +   迁移清单和 API 客户端以使用 `autoscaling/v2` API 版本，自 v1.23 起可用。
> +   
> +   所有现有的持久化对象可通过新的 API 访问
> +   
> 弃用 API 迁移指南

为使清单未来兼容，您只需分配新的 API 版本。

# 处理已移除或替换的 API

Kubernetes 集群的管理员可能会决定一次跳过多个小版本进行升级。可能您并未注意到当前正在使用的 API 已经被移除。在升级生产集群之前，验证现有 Kubernetes 对象的兼容性非常重要，以避免任何中断。

假设您一直使用 示例 13-2 中展示的 ClusterRole 的定义。在 Kubernetes 1.8 中管理该对象运作良好；然而，管理员将集群节点升级到了 1.22。

##### 示例 13-2\. 使用 API 版本 `rbac.authorization.k8s.io/v1beta1` 的 ClusterRole

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

尝试从 ClusterRole 清单文件中创建对象将不会显示弃用消息。相反，`kubectl` 将返回一个错误消息。因此，命令未创建对象：

```
$ kubectl apply -f clusterrole.yaml
error: resource mapping not found for name: "pod-reader" namespace: "" from \
"clusterrole.yaml": no matches for kind "ClusterRole" in version \
"rbac.authorization.k8s.io/v1beta1"

```

仅仅通过查看错误消息，你无法清楚地了解命令失败的原因。建议查阅弃用 API 迁移指南。在页面上搜索 API 版本 `rbac.authorization.k8s.io/v1beta1` 将给出 [以下信息](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#rbac-resources-v122)。这里的解决方案是改用替代 API `rbac.authorization.k8s.io/v1`：

> `rbac.authorization.k8s.io/v1beta1` API 版本的 ClusterRole、ClusterRoleBinding、Role 和 RoleBinding 已在 v1.22 中停止服务。
> 
> +   迁移清单和 API 客户端以使用 `rbac.authorization.k8s.io/v1` API 版本，自 v1.8 起可用。
> +   
> +   所有现有的持久化对象可通过新的 API 访问
> +   
> +   无显著变化
> +   
> 弃用 API 迁移指南

在某些条件下，Kubernetes 原语可能无法提供替代 API。作为一个典型的用例，我想提到 PodSecurityPolicy 原语。该功能已被一个新的 Kubernetes 内部概念所取代，即 [Pod Security Admission](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#psp-v125)。您应该查阅即将发布的 Kubernetes 版本的发布说明，以了解更加重大的变更。

# 总结

长期使用 Kubernetes 时，您将不可避免地遇到 API 弃用问题。作为开发者，您需要知道如何解释弃用警告消息。Kubernetes 文档提供了识别替代 API 或功能所需的所有信息。建议在将节点升级到新版本 Kubernetes 之前，测试所有当前生产集群中使用的 YAML 清单对象。

# 考试要点

随时保留 API 弃用文档

考试可能会使您遇到使用弃用 API 的对象。您需要随时保留 [弃用 API 迁移指南](https://kubernetes.io/docs/reference/using-api/deprecation-guide/) 文档页面，以应对此类情况。该页面按 Kubernetes 版本分类描述了已弃用、已移除和已替换的 API。使用浏览器的搜索功能快速查找有关 API 的相关信息。页面右侧的快速参考链接让您可以快速导航到特定的 Kubernetes 版本。

# 样例练习

这些练习的解决方案可以在 附录 A 中找到。

1.  负责集群的 Kubernetes 管理员计划将所有节点从 Kubernetes 1.8 升级到 1.28。您定义了多个 Kubernetes YAML 清单，用于操作一个应用程序堆栈。管理员为您提供了一个 Kubernetes 1.28 测试环境。确保所有 YAML 清单与 Kubernetes 版本 1.28 兼容。

    转到已检出 GitHub 仓库 [*bmuschko/ckad-study-guide*](https://github.com/bmuschko/ckad-study-guide) 的目录 *app-a/ch13/deprecated*。检查当前目录中的文件 *deployment.yaml* 和 *configmap.yaml*。

    根据 YAML 清单创建对象。根据需要修改定义。

    确保对象能在 Kubernetes 1.28 中实例化。
