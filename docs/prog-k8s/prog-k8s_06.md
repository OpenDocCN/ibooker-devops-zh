# 第六章：解决方案编写操作员

到目前为止，我们已经在“控制器和操作员”的概念层面上看过自定义控制器和操作员，以及在第五章中，如何使用 Kubernetes 代码生成器——这是一种处理该主题的较低级别方法。在本章中，我们将详细介绍三种编写自定义控制器和操作员的解决方案，并讨论一些更多的替代方法。

在本章讨论的解决方案中使用其中一个应该有助于您避免编写大量重复的代码，并使您能够专注于业务逻辑，而不是样板代码。这应该能够帮助您更快地入门并提高工作效率。

###### 注意

总体而言，操作员以及我们在本章中讨论的工具截至 2019 年中仍在快速发展。尽管我们尽力，但这里显示的某些命令和/或它们的输出可能会更改。请考虑到这一点，并确保您始终使用相应工具的最新版本，并关注相应的问题跟踪器、邮件列表和 Slack 频道。

尽管在线上有可[比较的资源](http://bit.ly/2ZC5fZT)可以比较我们在这里讨论的解决方案，但我们不会向您推荐特定的解决方案。不过，我们鼓励您自行评估和比较它们，并选择最适合您的组织和环境的解决方案。

# 准备工作

我们将使用`cnat`（云原生的 `at`，我们在“一个激励示例”中介绍过）作为本章中不同解决方案的运行示例。如果您想跟随我们一起学习，请注意我们假设您：

1.  已经安装并正确设置了 Go 版本 1.12 或更高版本。

1.  拥有一个 Kubernetes 集群，版本为 1.12 或更高版本——可以是本地的，例如通过`kind`或`k3d`，也可以是通过您喜欢的云提供商远程的，并且配置了`kubectl`以访问它。

1.  `git clone`我们的[GitHub 代码库](http://bit.ly/2N3R6U4)。完整、可运行的源代码以及以下部分显示的必要命令都可以在那里找到。请注意，我们这里展示的是从零开始的工作方式。如果您只想查看结果而不是自己执行步骤，请随时克隆代码库并仅运行安装 CRD、安装 CR 和启动自定义控制器的命令。

在处理完这些杂务事项后，让我们开始编写操作员吧：我们将在本章涵盖`sample-controller`、Kubebuilder 和 Operator SDK。

准备好了吗？我们开始吧——此处有双关语！

# 跟随`sample-controller`

让我们从基于[*k8s.io/sample-controller*](http://bit.ly/2UppsTN)实现`cnat`开始，它直接使用`client-go`库来实现。（请查看[*k8s.io/code-generator*](http://bit.ly/2Kw8I8U)用于生成类型化客户端、通知器、列表器和深拷贝函数。每当你的自定义控制器中的 API 类型发生变化，例如在自定义资源中添加一个新字段，你都必须使用*update-codegen.sh*脚本（还可以在 GitHub 中查看其[源码](http://bit.ly/2Fq3Td1)）重新生成上述源文件。

###### 警告

你可能已经注意到本书中始终使用*k8s.io*作为基础 URL。我们在第三章介绍了它的用法；作为提醒，它实际上是*kubernetes.io*的别名，在 Go 包管理的上下文中解析为*github.com/kubernetes*。请注意，*k8s.io*不具有自动重定向功能。例如，*k8s.io/sample-controller*实际上意味着你应该查看[*github.com/kubernetes/sample-controller*](http://bit.ly/2UppsTN)，等等。

好的，让我们使用[`cnat`](http://bit.ly/2RpHhON)操作符来实现，使用`client-go`，遵循`sample-controller`。（参见我们仓库中的[对应目录](http://bit.ly/2N3R6U4)。）

## 引导

要开始，请执行**`go get k8s.io/sample-controller`**以获取源码和依赖项到你的系统中，应该位于*$GOPATH/src/k8s.io/sample-\controller*目录中。

如果你从零开始，将*sample-controller*目录的内容复制到你选择的目录中（例如，在我们的仓库中使用*cnat-client-go*），然后可以运行以下命令序列来构建和运行基础控制器（默认实现，尚未涉及`cnat`业务逻辑）：

```
# build custom controller binary:
$ go build -o cnat-controller .

# launch custom controller locally:
$ ./cnat-controller -kubeconfig=$HOME/.kube/config
```

这个命令将启动自定义控制器，并等待你注册 CRD 并创建自定义资源。现在让我们来做这件事，并看看会发生什么。在第二个终端会话中，输入：

```
$ kubectl apply -f artifacts/examples/crd.yaml
```

确保 CRD 已正确注册并可用，如下所示：

```
$ kubectl get crds
NAME                            CREATED AT
foos.samplecontroller.k8s.io    2019-05-29T12:16:57Z
```

请注意，你可能会看到其他 CRD，这取决于你使用的 Kubernetes 发行版；但是，至少应该列出*foos.samplecontroller.k8s.io*。

接下来，我们创建示例自定义资源*foo.samplecontroller.k8s.io/example-foo*并检查控制器是否正常工作：

```
$ kubectl apply -f artifacts/examples/example-foo.yaml
foo.samplecontroller.k8s.io/example-foo created

$ kubectl get po,rs,deploy,foo
NAME                                           READY   STATUS    RESTARTS   AGE
pod/example-foo-5b8c9679d8-xjhdf               1/1     Running   0          67s

NAME                                           DESIRED   CURRENT   READY AGE
replicaset.extensions/example-foo-5b8c9679d8   1         1         1     67s

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/example-foo              1/1     1            1           67s

NAME                                           AGE
foo.samplecontroller.k8s.io/example-foo        67s
```

太棒了，它按预期工作！现在我们可以继续实现实际的`cnat`特定业务逻辑。

## 业务逻辑

要启动业务逻辑的实现，我们首先将现有的*pkg/apis/samplecontroller*目录重命名为*pkg/apis/cnat*，然后按以下步骤创建我们自己的 CRD 和自定义资源：

```
$ cat artifacts/examples/cnat-crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  version: v1alpha1
  names:
    kind: At
    plural: ats
  scope: Namespaced

$ cat artifacts/examples/cnat-example.yaml
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: example-at
spec:
  schedule: "2019-04-12T10:12:00Z"
  command: "echo YAY"
```

请注意，每当 API 类型发生变化时，例如当你向`At` CRD 中添加新字段时，你必须执行*update-codegen.sh*脚本，如下所示：

```
$ ./hack/update-codegen.sh
```

这将自动生成以下内容：

+   *pkg/apis/cnat/v1alpha1/zz_generated.deepcopy.go*

+   *pkg/generated/**

在业务逻辑方面，我们需要实现运算符中的两个部分：

+   在[*types.go*](http://bit.ly/31QosJw)中，我们修改了`AtSpec`结构体，包括相应的字段，如`schedule`和`command`。请注意，每当您在此处进行更改时，都必须运行`update-codegen.sh`以重新生成依赖文件。

+   在[*controller.go*](http://bit.ly/31MM4OS)中，我们修改了`NewController()`和`syncHandler()`函数，并添加了辅助函数，包括创建 pod 和检查调度时间。

在*types.go*中，请注意代表`At`资源三个阶段的三个常量：在`PENDING`状态中直到调度时间，然后`RUNNING`直至完成，最后处于`DONE`状态：

```
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be
    // executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is
    // executed it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

请注意，显式使用构建标签`+k8s:deepcopy-gen:interfaces`（参见第五章），以便自动生成相应的源代码。

现在我们可以开始实现自定义控制器的业务逻辑了。也就是说，我们在[controller.go](http://bit.ly/31MM4OS)中实现了三个阶段之间的状态转换——从`PhasePending`到`PhaseRunning`再到`PhaseDone`。

在“工作队列”中，我们介绍并解释了`client-go`提供的工作队列。现在我们可以将这些知识投入实践：在*controller.go*的`processNextWorkItem()`函数中——更确切地说，在[第 176 到 186 行](http://bit.ly/2WYDbyi)——您可以找到以下（生成的）代码：

```
if when, err := c.syncHandler(key); err != nil {
    c.workqueue.AddRateLimited(key)
    return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
} else if when != time.Duration(0) {
    c.workqueue.AddAfter(key, when)
} else {
    // Finally, if no error occurs we Forget this item so it does not
    // get queued again until another change happens.
    c.workqueue.Forget(obj)
}
```

此片段展示了我们（尚未编写的）自定义`syncHandler()`函数（稍后解释）如何被调用并涵盖这三种情况：

1.  第一个`if`分支通过调用`AddRateLimited()`函数重新排队项目，处理瞬态错误。

1.  第二个分支，即`else if`，通过调用`AddAfter()`函数重新排队项目，以避免热循环。

1.  最后一个情况，即`else`，是项目已成功处理并通过调用`Forget()`函数丢弃。

现在我们已经对通用处理有了清晰的理解，让我们继续讨论业务逻辑特定的功能。其中关键的是前述的`syncHandler()`函数，我们在这里实现了自定义控制器的业务逻辑。其函数签名如下：

```
// syncHandler compares the actual state with the desired state and attempts
// to converge the two. It then updates the Status block of the At resource
// with the current status of the resource. It returns how long to wait
// until the schedule is due.
func (c *Controller) syncHandler(key string) (time.Duration, error) {
    ...
}
```

此`syncHandler()`函数实现了以下状态转换：^(1)

```
...
// If no phase set, default to pending (the initial phase):
if instance.Status.Phase == "" {
    instance.Status.Phase = cnatv1alpha1.PhasePending
}

// Now let's make the main case distinction: implementing
// the state diagram PENDING -> RUNNING -> DONE
switch instance.Status.Phase {
case cnatv1alpha1.PhasePending:
    klog.Infof("instance %s: phase=PENDING", key)
    // As long as we haven't executed the command yet, we need
    // to check if it's time already to act:
    klog.Infof("instance %s: checking schedule %q", key, instance.Spec.Schedule)
    // Check if it's already time to execute the command with a
    // tolerance of 2 seconds:
    d, err := timeUntilSchedule(instance.Spec.Schedule)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("schedule parsing failed: %v", err))
        // Error reading the schedule - requeue the request:
        return time.Duration(0), err
    }
    klog.Infof("instance %s: schedule parsing done: diff=%v", key, d)
    if d > 0 {
        // Not yet time to execute the command, wait until the
        // scheduled time
        return d, nil
    }

    klog.Infof(
       "instance %s: it's time! Ready to execute: %s", key,
       instance.Spec.Command,
    )
    instance.Status.Phase = cnatv1alpha1.PhaseRunning
case cnatv1alpha1.PhaseRunning:
    klog.Infof("instance %s: Phase: RUNNING", key)

    pod := newPodForCR(instance)

    // Set At instance as the owner and controller
    owner := metav1.NewControllerRef(
        instance, cnatv1alpha1.SchemeGroupVersion.
        WithKind("At"),
    )
    pod.ObjectMeta.OwnerReferences = append(pod.ObjectMeta.OwnerReferences, *owner)

    // Try to see if the pod already exists and if not
    // (which we expect) then create a one-shot pod as per spec:
    found, err := c.kubeClientset.CoreV1().Pods(pod.Namespace).
        Get(pod.Name, metav1.GetOptions{})
    if err != nil && errors.IsNotFound(err) {
        found, err = c.kubeClientset.CoreV1().Pods(pod.Namespace).Create(pod)
        if err != nil {
            return time.Duration(0), err
        }
        klog.Infof("instance %s: pod launched: name=%s", key, pod.Name)
    } else if err != nil {
        // requeue with error
        return time.Duration(0), err
    } else if found.Status.Phase == corev1.PodFailed ||
        found.Status.Phase == corev1.PodSucceeded {
        klog.Infof(
            "instance %s: container terminated: reason=%q message=%q",
            key, found.Status.Reason, found.Status.Message,
        )
        instance.Status.Phase = cnatv1alpha1.PhaseDone
    } else {
        // Don't requeue because it will happen automatically
        // when the pod status changes.
        return time.Duration(0), nil
    }
case cnatv1alpha1.PhaseDone:
    klog.Infof("instance %s: phase: DONE", key)
    return time.Duration(0), nil
default:
    klog.Infof("instance %s: NOP")
    return time.Duration(0), nil
}

// Update the At instance, setting the status to the respective phase:
_, err = c.cnatClientset.CnatV1alpha1().Ats(instance.Namespace).
    UpdateStatus(instance)
if err != nil {
    return time.Duration(0), err
}

// Don't requeue. We should be reconcile because either the pod or
// the CR changes.
return time.Duration(0), nil
```

此外，为了设置 informer 和整个控制器，我们在`NewController()`中实现了以下内容：

```
// NewController returns a new cnat controller
func NewController(
    kubeClientset kubernetes.Interface,
    cnatClientset clientset.Interface,
    atInformer informers.AtInformer,
    podInformer corev1informer.PodInformer) *Controller {

    // Create event broadcaster
    // Add cnat-controller types to the default Kubernetes Scheme so Events
    // can be logged for cnat-controller types.
    utilruntime.Must(cnatscheme.AddToScheme(scheme.Scheme))
    klog.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartLogging(klog.Infof)
    eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{
        Interface: kubeClientset.CoreV1().Events(""),
    })
    source := corev1.EventSource{Component: controllerAgentName}
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme, source)

    rateLimiter := workqueue.DefaultControllerRateLimiter()
    controller := &Controller{
        kubeClientset: kubeClientset,
        cnatClientset: cnatClientset,
        atLister:      atInformer.Lister(),
        atsSynced:     atInformer.Informer().HasSynced,
        podLister:     podInformer.Lister(),
        podsSynced:    podInformer.Informer().HasSynced,
        workqueue:     workqueue.NewNamedRateLimitingQueue(rateLimiter, "Ats"),
        recorder:      recorder,
    }

    klog.Info("Setting up event handlers")
    // Set up an event handler for when At resources change
    atInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueAt,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueueAt(new)
        },
    })
    // Set up an event handler for when Pod resources change
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueuePod,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueuePod(new)
        },
    })
    return controller
}
```

我们还需要两个进一步的辅助函数来使其工作：一个计算直到调度时间的时间，形式如下：

```
func timeUntilSchedule(schedule string) (time.Duration, error) {
    now := time.Now().UTC()
    layout := "2006-01-02T15:04:05Z"
    s, err := time.Parse(layout, schedule)
    if err != nil {
        return time.Duration(0), err
    }
    return s.Sub(now), nil
}
```

另外一个则使用`busybox`容器镜像创建一个带有执行命令的 pod：

```
func newPodForCR(cr *cnatv1alpha1.At) *corev1.Pod {
    labels := map[string]string{
        "app": cr.Name,
    }
    return &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cr.Name + "-pod",
            Namespace: cr.Namespace,
            Labels:    labels,
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:    "busybox",
                    Image:   "busybox",
                    Command: strings.Split(cr.Spec.Command, " "),
                },
            },
            RestartPolicy: corev1.RestartPolicyOnFailure,
        },
    }
}
```

我们将在本章后面的`syncHandler()`函数中重复使用这两个辅助函数和业务逻辑的基本流程，请确保您熟悉它们的详细内容。

注意，从 `At` 资源的角度来看，Pod 是一个次要资源，控制器必须确保清理这些 Pod，否则可能导致孤立的 Pod。

现在，`sample-controller` 是一个学习如何制作香肠的好工具，但通常情况下，您希望专注于创建业务逻辑，而不是处理样板代码。为此，您可以选择两个相关项目：Kubebuilder 和 Operator SDK。让我们分别看看它们以及如何使用它们实现 `cnat`。

# Kubebuilder

[Kubebuilder](http://bit.ly/2I8w9mz)，由 Kubernetes 特别兴趣小组（SIG）API Machinery 拥有和维护，是一款工具和一组库，能够让您以简单高效的方式构建运算符。深入了解 Kubebuilder 最佳资源是在线的 [Kubebuilder 书籍](https://book.kubebuilder.io)，该书籍详细介绍了其组件和用法。不过，我们在这里将专注于如何使用 Kubebuilder 实现我们的 [`cnat`](http://bit.ly/2RpHhON) 运算符（请参阅我们 Git 仓库中的 [对应目录](http://bit.ly/2Iv6pAS)）。

首先，让我们确保已安装所有依赖项，即 [dep](http://bit.ly/2x9Yrqq)、[kustomize](http://bit.ly/2Y3JeCV)（参见 “Kustomize”）以及 [Kubebuilder 本身](http://bit.ly/32pQmfu)：

```
$ dep version
dep:
 version     : v0.5.1
 build date  : 2019-03-11
 git hash    : faa6189
 go version  : go1.12
 go compiler : gc
 platform    : darwin/amd64
 features    : ImportDuringSolve=false

$ kustomize version
Version: {KustomizeVersion:v2.0.3 GitCommit:a6f65144121d1955266b0cd836ce954c04122dc8
          BuildDate:2019-03-18T22:15:21+00:00 GoOs:darwin GoArch:amd64}

$ kubebuilder version
Version: version.Version{
  KubeBuilderVersion:"1.0.8",
  KubernetesVendor:"1.13.1",
  GitCommit:"1adf50ed107f5042d7472ba5ab50d5e1d357169d",
  BuildDate:"2019-01-25T23:14:29Z", GoOs:"unknown", GoArch:"unknown"
}
```

我们将逐步指导您从头开始编写 `cnat` 运算符的步骤。首先，在您选择的目录（我们在我们的仓库中使用 *cnat-kubebuilder*）中创建基础目录，这将作为以后所有命令的基础。

###### 警告

在撰写本文时，Kubebuilder 正在迁移到新版本（v2）。由于尚未稳定，我们展示的是（稳定的）[版本 v1](https://book-v1.book.kubebuilder.io) 的命令和设置。

## 启动

要启动 `cnat` 运算符，我们使用 `init` 命令，如下（请注意，根据您的环境，此过程可能需要几分钟）：

```
$ kubebuilder init \
              --domain programming-kubernetes.info \
              --license apache2 \
              --owner "Programming Kubernetes authors"
Run `dep ensure` to fetch dependencies (Recommended) [y/n]?
y
dep ensure
Running make...
make
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
go test ./pkg/... ./cmd/... -coverprofile cover.out
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis        [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/controller  [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/webhook     [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/cmd/manager     [no test files]
go build -o bin/manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

完成此命令后，Kubebuilder 将生成一堆文件，从自定义控制器到示例 CRD。您的基础目录现在应该看起来像下面这样（为清晰起见，不包括庞大的 *vendor* 目录）：

```
$ tree -I vendor
.
├── Dockerfile
├── Gopkg.lock
├── Gopkg.toml
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── cmd
│   └── manager
│       └── main.go
├── config
│   ├── crds
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_image_patch.yaml
│   │   └── manager_prometheus_metrics_patch.yaml
│   ├── manager
│   │   └── manager.yaml
│   └── rbac
│       ├── auth_proxy_role.yaml
│       ├── auth_proxy_role_binding.yaml
│       ├── auth_proxy_service.yaml
│       ├── rbac_role.yaml
│       └── rbac_role_binding.yaml
├── cover.out
├── hack
│   └── boilerplate.go.txt
└── pkg
    ├── apis
    │   └── apis.go
    ├── controller
    │   └── controller.go
    └── webhook
        └── webhook.go

13 directories, 22 files
```

接下来，我们使用 `create api` 命令创建 API，也就是自定义控制器（这比之前的命令更快，但仍需一段时间）：

```
$ kubebuilder create api \
              --group cnat \
              --version v1alpha1 \
              --kind At
Create Resource under pkg/apis [y/n]?
y
Create Controller under pkg/controller [y/n]?
y
Writing scaffold for you to edit...
pkg/apis/cnat/v1alpha1/at_types.go
pkg/apis/cnat/v1alpha1/at_types_test.go
pkg/controller/at/at_controller.go
pkg/controller/at/at_controller_test.go
Running make...
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
go test ./pkg/... ./cmd/... -coverprofile cover.out
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis        [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat   [no test files]
ok      github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat/v1alpha1  9.011s
?       github.com/mhausenblas/cnat-kubebuilder/pkg/controller  [no test files]
ok      github.com/mhausenblas/cnat-kubebuilder/pkg/controller/at       8.740s
?       github.com/mhausenblas/cnat-kubebuilder/pkg/webhook     [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/cmd/manager     [no test files]
go build -o bin/manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

让我们来看看有哪些变化，重点放在两个已更新并添加内容的目录上：

```
$ tree config/ pkg/
config/
├── crds
│   └── cnat_v1alpha1_at.yaml
├── default
│   ├── kustomization.yaml
│   ├── manager_auth_proxy_patch.yaml
│   ├── manager_image_patch.yaml
│   └── manager_prometheus_metrics_patch.yaml
├── manager
│   └── manager.yaml
├── rbac
│   ├── auth_proxy_role.yaml
│   ├── auth_proxy_role_binding.yaml
│   ├── auth_proxy_service.yaml
│   ├── rbac_role.yaml
│   └── rbac_role_binding.yaml
└── samples
    └── cnat_v1alpha1_at.yaml
pkg/
├── apis
│   ├── addtoscheme_cnat_v1alpha1.go
│   ├── apis.go
│   └── cnat
│       ├── group.go
│       └── v1alpha1
│           ├── at_types.go
│           ├── at_types_test.go
│           ├── doc.go
│           ├── register.go
│           ├── v1alpha1_suite_test.go
│           └── zz_generated.deepcopy.go
├── controller
│   ├── add_at.go
│   ├── at
│   │   ├── at_controller.go
│   │   ├── at_controller_suite_test.go
│   │   └── at_controller_test.go
│   └── controller.go
└── webhook
    └── webhook.go

11 directories, 27 files
```

注意在 *config/crds/* 中新增了 *cnat_v1alpha1_at.yaml*，这是自定义资源定义（CRD）的一部分；同时，在 *config/samples/* 中也有一个名为 *cnat_v1alpha1_at.yaml* 的文件（是的，同名），表示了该 CRD 的一个示例实例。此外，在 *pkg/* 目录下我们看到了一些新文件，其中最重要的是 *apis/cnat/v1alpha1/at_types.go* 和 *controller/at/at_controller.go*，我们接下来将修改它们。

接下来，我们在 Kubernetes 中创建一个专用的命名空间`cnat`，并将其用作默认命名空间，设置上下文如下（作为一个良好的实践，始终使用专用命名空间而不是`default`）：

```
$ kubectl create ns cnat && \
  kubectl config set-context $(kubectl config current-context) --namespace=cnat
```

我们使用以下命令安装 CRD：

```
$ make install
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
kubectl apply -f config/crds
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info created
```

现在我们可以在本地启动操作者：

```
$ make run
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run ./cmd/manager/main.go
{"level":"info","ts":1559152740.0550249,"logger":"entrypoint",
  "msg":"setting up client for manager"}
{"level":"info","ts":1559152740.057556,"logger":"entrypoint",
  "msg":"setting up manager"}
{"level":"info","ts":1559152740.1396701,"logger":"entrypoint",
  "msg":"Registering Components."}
{"level":"info","ts":1559152740.1397,"logger":"entrypoint",
  "msg":"setting up scheme"}
{"level":"info","ts":1559152740.139773,"logger":"entrypoint",
  "msg":"Setting up controller"}
{"level":"info","ts":1559152740.139831,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1559152740.139929,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1559152740.139971,"logger":"entrypoint",
  "msg":"setting up webhooks"}
{"level":"info","ts":1559152740.13998,"logger":"entrypoint",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1559152740.244628,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1559152740.344791,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

将终端会话保持运行状态，并在新会话中安装 CRD，验证它，并像这样创建样本自定义资源：

```
$ kubectl apply -f config/crds/cnat_v1alpha1_at.yaml
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info
configured

$ kubectl get crds
NAME                                   CREATED AT
ats.cnat.programming-kubernetes.info   2019-05-29T17:54:51Z

$ kubectl apply -f config/samples/cnat_v1alpha1_at.yaml
at.cnat.programming-kubernetes.info/at-sample created
```

如果您现在查看`make run`运行的会话输出，您应该注意到以下输出：

```
...
{"level":"info","ts":1559153311.659829,"logger":"controller",
  "msg":"Creating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.678407,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.6839428,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.693443,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.7023401,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153332.986961,"logger":"controller",#
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
```

这告诉我们整体设置是成功的！既然我们完成了搭建并成功启动了`cnat`操作者，现在可以继续执行实际的核心任务：使用 Kubebuilder 实现`cnat`业务逻辑。

## 业务逻辑

首先，我们将更改[*config/crds/cnat_v1alpha1_at.yaml*](http://bit.ly/2N1jQNb)和[*config/samples/cnat_v1alpha1_at.yaml*](http://bit.ly/2Xs1F7c)以适应我们自己的`cnat` CRD 和自定义资源值定义，重用与“Following sample-controller”中相同的结构。

就业务逻辑而言，操作者有两个部分需要实现：

+   在[*pkg/apis/cnat/v1alpha1/at_types.go*](http://bit.ly/31KNLfO)中，我们修改了`AtSpec`结构体，包括相关字段如`schedule`和`command`。请注意，每当您在这里更改了某些内容时，都必须运行`make`来重新生成依赖文件。Kubebuilder 使用 Kubernetes 生成器（在第五章中描述）并提供自己的生成器集（例如，用于生成 CRD 清单）。

+   在[*pkg/controller/at/at_controller.go*](http://bit.ly/2Iwormg)中，我们修改`Reconcile(request reconcile.Request)`方法以在`Spec.Schedule`定义的时间创建一个 Pod。

在*at_types.go*中：

```
const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在*at_controller.go*中，我们实现了三个阶段之间的状态转换，从`PENDING`到`RUNNING`再到`DONE`：

```
func (r *ReconcileAt) Reconcile(req reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("namespace", req.Namespace, "at", req.Name)
    reqLogger.Info("=== Reconciling At")
    // Fetch the At instance
    instance := &cnatv1alpha1.At{}
    err := r.Get(context.TODO(), req.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after
            // reconcile request—return and don't requeue:
            return reconcile.Result{}, nil
        }
            // Error reading the object—requeue the request:
        return reconcile.Result{}, err
    }

    // If no phase set, default to pending (the initial phase):
    if instance.Status.Phase == "" {
        instance.Status.Phase = cnatv1alpha1.PhasePending
    }

    // Now let's make the main case distinction: implementing
    // the state diagram PENDING -> RUNNING -> DONE
    switch instance.Status.Phase {
    case cnatv1alpha1.PhasePending:
        reqLogger.Info("Phase: PENDING")
        // As long as we haven't executed the command yet, we need to check if
        // it's already time to act:
        reqLogger.Info("Checking schedule", "Target", instance.Spec.Schedule)
        // Check if it's already time to execute the command with a tolerance
        // of 2 seconds:
        d, err := timeUntilSchedule(instance.Spec.Schedule)
        if err != nil {
            reqLogger.Error(err, "Schedule parsing failure")
            // Error reading the schedule. Wait until it is fixed.
            return reconcile.Result{}, err
        }
        reqLogger.Info("Schedule parsing done", "Result", "diff",
            fmt.Sprintf("%v", d))
        if d > 0 {
            // Not yet time to execute the command, wait until the scheduled time
            return reconcile.Result{RequeueAfter: d}, nil
        }
        reqLogger.Info("It's time!", "Ready to execute", instance.Spec.Command)
        instance.Status.Phase = cnatv1alpha1.PhaseRunning
    case cnatv1alpha1.PhaseRunning:
        reqLogger.Info("Phase: RUNNING")
        pod := newPodForCR(instance)
        // Set At instance as the owner and controller
        err := controllerutil.SetControllerReference(instance, pod, r.scheme)
        if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        }
        found := &corev1.Pod{}
        nsName := types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}
        err = r.Get(context.TODO(), nsName, found)
        // Try to see if the pod already exists and if not
        // (which we expect) then create a one-shot pod as per spec:
        if err != nil && errors.IsNotFound(err) {
            err = r.Create(context.TODO(), pod)
            if err != nil {
            // requeue with error
                return reconcile.Result{}, err
            }
            reqLogger.Info("Pod launched", "name", pod.Name)
        } else if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        } else if found.Status.Phase == corev1.PodFailed ||
                  found.Status.Phase == corev1.PodSucceeded {
            reqLogger.Info("Container terminated", "reason",
                found.Status.Reason, "message", found.Status.Message)
            instance.Status.Phase = cnatv1alpha1.PhaseDone
        } else {
            // Don't requeue because it will happen automatically when the
            // pod status changes.
            return reconcile.Result{}, nil
        }
    case cnatv1alpha1.PhaseDone:
        reqLogger.Info("Phase: DONE")
        return reconcile.Result{}, nil
    default:
        reqLogger.Info("NOP")
        return reconcile.Result{}, nil
    }

    // Update the At instance, setting the status to the respective phase:
    err = r.Status().Update(context.TODO(), instance)
    if err != nil {
        return reconcile.Result{}, err
    }

    // Don't requeue. We should be reconcile because either the pod
    // or the CR changes.
    return reconcile.Result{}, nil
}
```

注意在最后的`Update`调用中，操作的是*/status*子资源（参见“状态子资源”），而不是整个 CR。因此，我们遵循了规范状态分离的最佳实践。

现在，一旦创建了 CR `example-at`，我们就会看到本地执行操作者的以下输出：

```
$ make run
...
{"level":"info","ts":1555063897.488535,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063897.488621,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063897.4886441,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063897.488703,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 22.511336s"}
{"level":"info","ts":1555063907.489264,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063907.489402,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063907.489428,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063907.489486,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 12.510551s"}
{"level":"info","ts":1555063917.490178,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063917.4902349,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063917.490247,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063917.490278,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 2.509743s"}
{"level":"info","ts":1555063927.492718,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.49283,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.492857,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063927.492915,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of -7.492877s"}
{"level":"info","ts":1555063927.4929411,"logger":"controller",
  "msg":"It's time!","namespace":"cnat","at":
  "example-at","Ready to execute":"echo YAY"}
{"level":"info","ts":1555063927.626236,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.626303,"logger":"controller",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063928.07445,"logger":"controller",
  "msg":"Pod launched","namespace":"cnat","at":"example-at",
  "name":"example-at-pod"}
{"level":"info","ts":1555063928.199562,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063928.199645,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631733,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631783,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

要验证我们的自定义控制器是否完成了其工作，请执行：

```
$ kubectl get at,pods
NAME                                                  AGE
at.cnat.programming-kubernetes.info/example-at        11m

NAME                 READY   STATUS        RESTARTS   AGE
pod/example-at-pod   0/1     Completed     0          38s
```

太棒了！`example-at-pod`已经创建，现在是时候看操作的结果了：

```
$ kubectl logs example-at-pod
YAY
```

当您完成开发自定义控制器时，使用如此展示的本地模式时，您可能希望构建一个容器镜像。此自定义控制器容器镜像随后可以在 Kubernetes 部署中使用，例如。您可以使用以下命令生成容器镜像并将其推送到仓库*quay.io/pk/cnat*：

```
$ export IMG=quay.io/pk/cnat:v1

$ make docker-build

$ make docker-push
```

接下来我们转向 Operator SDK，它与 Kubebuilder 的代码库和 API 有些共享。

# Operator SDK

为了更容易构建 Kubernetes 应用程序，CoreOS/Red Hat 组合了 Operator Framework。 其中的一部分是 [Operator SDK](http://bit.ly/2KtpK7D)，它使开发人员能够构建操作者，而无需深入了解 Kubernetes API。

Operator SDK 提供了构建、测试和打包操作者的工具。 虽然 SDK 提供了更多功能，特别是关于测试的功能，但我们在这里专注于使用 SDK 实现我们的 [`cnat`](http://bit.ly/2RpHhON) 操作者（参见 [我们 Git 仓库中对应的目录](http://bit.ly/2FpCtE9)）。

首先确保 [安装 Operator SDK](http://bit.ly/2ZBQlCT)，并检查所有依赖项是否可用：

```
$ dep version
dep:
 version     : v0.5.1
 build date  : 2019-03-11
 git hash    : faa6189
 go version  : go1.12
 go compiler : gc
 platform    : darwin/amd64
 features    : ImportDuringSolve=false

 $ operator-sdk --version
operator-sdk version v0.6.0
```

## 引导

现在是时候引导 `cnat` 操作者了：

```
$ operator-sdk new cnat-operator && cd cnat-operator
```

接下来，与 Kubebuilder 非常相似，我们添加一个 API——或者简单地说：初始化自定义控制器，如下所示：

```
$ operator-sdk add api \
               --api-version=cnat.programming-kubernetes.info/v1alpha1 \
               --kind=At

$ operator-sdk add controller \
               --api-version=cnat.programming-kubernetes.info/v1alpha1 \
               --kind=At
```

这些命令生成了必要的样板代码，以及一些辅助函数，例如深度复制函数 `DeepCopy()`、`DeepCopyInto()` 和 `DeepCopyObject()`。

现在我们可以将自动生成的 CRD 应用到 Kubernetes 集群中：

```
$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_crd.yaml

$ kubectl get crds
NAME                                             CREATED AT
ats.cnat.programming-kubernetes.info             2019-04-01T14:03:33Z
```

让我们在本地启动我们的 `cnat` 自定义控制器。 通过这样做，它可以开始处理请求：

```
$ OPERATOR_NAME=cnatop operator-sdk up local --namespace "cnat"
INFO[0000] Running the operator locally.
INFO[0000] Using namespace cnat.
{"level":"info","ts":1555041531.871706,"logger":"cmd",
  "msg":"Go Version: go1.12.1"}
{"level":"info","ts":1555041531.871785,"logger":"cmd",
  "msg":"Go OS/Arch: darwin/amd64"}
{"level":"info","ts":1555041531.8718028,"logger":"cmd",
  "msg":"Version of operator-sdk: v0.6.0"}
{"level":"info","ts":1555041531.8739321,"logger":"leader",
  "msg":"Trying to become the leader."}
{"level":"info","ts":1555041531.8743382,"logger":"leader",
  "msg":"Skipping leader election; not running in a cluster."}
{"level":"info","ts":1555041536.1611362,"logger":"cmd",
  "msg":"Registering Components."}
{"level":"info","ts":1555041536.1622112,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1555041536.162519,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1555041539.978822,"logger":"metrics",
  "msg":"Skipping metrics Service creation; not running in a cluster."}
{"level":"info","ts":1555041539.978875,"logger":"cmd",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1555041540.179469,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1555041540.280784,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

我们的自定义控制器将保持在此状态，直到我们创建了 CR *ats.cnat.programming-kubernetes.info*。 那么让我们开始吧：

```
$ cat deploy/crds/cnat_v1alpha1_at_cr.yaml
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-04-11T14:56:30Z"
  command: "echo YAY"

$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_cr.yaml

$ kubectl get at
NAME                                             AGE
at.cnat.programming-kubernetes.info/example-at   54s
```

## 业务逻辑

在业务逻辑方面，我们需要实现操作者的两个部分：

+   在 [*pkg/apis/cnat/v1alpha1/at_types.go*](http://bit.ly/31Ip2sF) 中，我们修改了 `AtSpec` 结构以包括相应的字段，如 `schedule` 和 `command`，并使用 `operator-sdk generate k8s` 重新生成代码，以及使用 `operator-sdk generate openapi` 命令处理 OpenAPI 部分。

+   在 [*pkg/controller/at/at_controller.go*](http://bit.ly/2Fpo5Mi) 中，我们修改了 `Reconcile(request reconcile.Request)` 方法，以在 `Spec.Schedule` 定义的时间创建一个 pod。

更详细地查看引导代码中的更改（专注于相关部分）。 在 *at_types.go* 中：

```
// AtSpec defines the desired state of At
// +k8s:openapi-gen=true
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
// +k8s:openapi-gen=true
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在 *at_controller.go* 中，我们实现了三个阶段的状态图，从 `PENDING` 到 `RUNNING` 到 `DONE`。

###### 注意

[`controller-runtime`](http://bit.ly/2ZFtDKd) 是另一个 SIG API Machinery 拥有的项目，旨在提供一套通用的低级功能，用于以 Go 包的形式构建控制器。 更多细节请参见 第四章。

由于 Kubebuilder 和 Operator SDK 都使用控制器运行时，`Reconcile()` 函数实际上是相同的：

```
func (r *ReconcileAt) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    *`the``-``same``-``as``-``for``-``kubebuilder`*
}

```

一旦创建了 CR `example-at`，我们就可以看到本地执行操作者的以下输出：

```
$ OPERATOR_NAME=cnatop operator-sdk up local --namespace "cnat"
INFO[0000] Running the operator locally.
INFO[0000] Using namespace cnat.
...
{"level":"info","ts":1555044934.023597,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.023713,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.0237482,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":
  "example-at","Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044934.02382,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 25.976236s"}
{"level":"info","ts":1555044934.148148,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.148224,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.148243,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044934.1482902,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 25.85174s"}
{"level":"info","ts":1555044944.1504588,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044944.150568,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044944.150599,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044944.150663,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 15.84938s"}
{"level":"info","ts":1555044954.385175,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044954.3852649,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044954.385288,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044954.38534,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 5.614691s"}
{"level":"info","ts":1555044964.518383,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.5184839,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.518566,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044964.5186381,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of -4.518596s"}
{"level":"info","ts":1555044964.5186849,"logger":"controller_at",
  "msg":"It's time!","namespace":"cnat","at":"example-at",
  "Ready to execute":"echo YAY"}
{"level":"info","ts":1555044964.642559,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.642622,"logger":"controller_at",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.911037,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.9111192,"logger":"controller_at",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038684,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038771,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708663,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708749,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

在这里，您可以看到我们操作者的三个阶段：从 `PENDING` 到时间戳 `1555044964.518566`，然后到 `RUNNING`，最后到 `DONE`。

要验证我们自定义控制器的功能并检查操作结果，请输入：

```
$ kubectl get at,pods
NAME                                                  AGE
at.cnat.programming-kubernetes.info/example-at        23m

NAME                 READY   STATUS        RESTARTS   AGE
pod/example-at-pod   0/1     Completed     0          46s

$ kubectl logs example-at-pod
YAY
```

当您完成开发自定义控制器，并像这里展示的那样使用本地模式时，您可能希望构建一个容器镜像。这个自定义控制器容器镜像随后可以用于例如 Kubernetes 部署。您可以使用以下命令生成容器镜像：

```
$ operator-sdk build $REGISTRY/PROJECT/IMAGE
```

这里有更多关于 Operator SDK 和相关示例的资源：

+   Toader Sebastian 在 BanzaiCloud 上的 [“Kubernetes 操作员 SDK 完全指南”](http://bit.ly/2RqkGSf)

+   Rob Szumski 的博客文章 [“为 Prometheus 和 Thanos 构建 Kubernetes 操作员”](http://bit.ly/2KvgHmu)

+   来自 CloudARK 在 ITNEXT 上的 [“提升可用性的 Kubernetes 操作员开发指南”](http://bit.ly/31P7rPC)

结束本章时，让我们看看编写自定义控制器和操作器的一些替代方法。

# 其他方法

除了或可能与我们讨论过的方法结合使用外，您可能还想看看以下项目、库和工具：

[元控制器](https://metacontroller.app)

Metacontroller 的基本思想是为您提供状态和变更的声明性规范，与基于水平触发的协调循环相接口，JSON 为基础。也就是说，您会收到描述观察状态的 JSON，并返回描述期望状态的 JSON。这对于在动态脚本语言（如 Python 或 JavaScript）中快速开发自动化特别有用。除了简单的控制器外，Metacontroller 还允许您将 API 组合成更高级的抽象，例如[BlueGreenDeployment](http://bit.ly/31KNTfi)。

[KUDO](https://kudo.dev)

类似于 Metacontroller，KUDO 提供了一个声明性方法来构建 Kubernetes 操作员，涵盖整个应用程序生命周期。简而言之，这是 Mesosphere 在 Apache Mesos 框架中的经验移植到 Kubernetes 中。KUDO 具有很强的见解性，但也很容易使用，几乎不需要编码；基本上，您只需指定一组带有定义执行逻辑的 Kubernetes 清单即可。

[Rook 操作器套件](http://bit.ly/2J34faw)

这是一个用于实现操作器的常见库。它起源于 Rook 操作器，但已经独立成为一个单独的项目。

[ericchiang/k8s](http://bit.ly/2ZHc5h0)

这是由 Eric Chiang 创造的一个简化的 Go 客户端，使用 Kubernetes 协议缓冲支持生成。它的行为类似于官方的 Kubernetes `client-go`，但只导入了两个外部依赖项。尽管它在某些方面存在限制，例如在[集群访问配置](http://bit.ly/2ZBQIxh)方面，但它是一个易于使用的简单 Go 包。

[`kutil`](http://bit.ly/2Fq3ojh)

AppsCode 通过 `kutil` 提供 Kubernetes `client-go` 的附加组件。

基于 CLI 客户端的方法

一种客户端方法，主要用于实验和测试，是以编程方式利用 `kubectl`（例如，[kubecuddler](http://bit.ly/2L3CDoi) 库）。

###### 注意

虽然本书侧重于使用 Go 编程语言编写运算符，但你也可以用其他语言编写运算符。两个显著的例子是 Flant 的[Shell-operator](http://bit.ly/2ZxkZ0m)，允许你使用经典的 Shell 脚本编写运算符，以及 Zalando 的[Kopf（Kubernetes 运算符框架）](http://bit.ly/2WRXU6Q)，一个 Python 框架和库。

如本章开头所述，运算符领域正在迅速发展，越来越多的从业者通过代码和最佳实践分享他们的知识，因此请密切关注这里的新工具。确保查阅在线资源和论坛，例如 Kubernetes Slack 上的`#kubernetes-operators`，`#kubebuilder`和`#client-go-docs`频道，以了解新方法和/或讨论问题，在遇到困难时寻求帮助。

# 采用和未来方向

到底哪种编写运算符的方法最受欢迎和广泛使用，目前还没有定论。在 Kubernetes 项目的背景下，涉及 CRs 和控制器的多个 SIGs 活动。主要的利益相关者是 SIG [API Machinery](http://bit.ly/2RuTPEp)，拥有 CRs 和控制器，并负责[Kubebuilder](http://bit.ly/2I8w9mz)项目。Operator SDK 已经加大了与 Kubebuilder API 对齐的努力，因此两者有很多重叠之处。

# 摘要

在本章中，我们看了几种不同的工具，让你能更高效地编写自定义控制器和运算符。传统上，只有跟随`sample-controller`是唯一的选择，但是有了 Kubebuilder 和 Operator SDK，你现在有了两个选项，可以让你专注于自定义控制器的业务逻辑，而不必处理样板文件。而且幸运的是，这两个工具共享了很多 API 和代码，所以从一个工具转换到另一个工具不应该太困难。

现在，让我们看看如何交付我们的劳动成果，即如何打包和发布我们一直在编写的控制器。

^(1) 这里仅展示相关部分；函数本身有很多其他的样板代码与我们的目的无关。
