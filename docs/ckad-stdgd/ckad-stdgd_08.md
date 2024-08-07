# 第八章：多容器 Pod

第五章 解释了如何管理单容器 Pod。这是正常情况，因为你希望在一个单独的 Pod 中运行一个微服务，以加强关注点的分离并增加内聚性。从技术上讲，一个 Pod 允许你配置和运行多个容器。

在本章中，我们将讨论多容器 Pod 的需求、相关的用例以及 Kubernetes 社区中出现的设计模式。考试大纲特别提到了几种主要的设计模式：初始化容器、旁车容器等。我们将通过代表性示例深入了解它们的应用。

# 在 Pod 中使用多个容器

特别是对于 Kubernetes 初学者来说，如何适当地设计一个 Pod 并不一定显而易见。如果你阅读 Kubernetes 用户文档和互联网上的教程，你很快就会发现，你可以创建一个同时运行多个容器的 Pod。那么问题来了：“我应该将我的微服务堆栈部署到一个具有多个容器的单个 Pod 中，还是应该创建多个每个运行一个微服务的 Pod？”简短的答案是每个 Pod 运行一个微服务。这种操作模式促进了分散化、解耦和分布式架构。此外，它有助于在不必中断系统其他部分的情况下推出微服务的新版本。

那么，在一个 Pod 中运行多个容器有什么意义呢？有两种常见的用例。有时，你希望通过执行设置脚本、命令或任何其他类型的预配置过程来初始化你的 Pod，这些逻辑在初始化容器中运行。其他时候，你可能希望提供与应用程序容器并行运行的辅助功能，以避免将逻辑嵌入到应用程序代码中。例如，你可能希望处理应用程序产生的日志输出。运行辅助逻辑的容器被称为*旁车容器*。

## 初始化容器

*初始化容器* 提供了在主应用程序启动之前运行的初始化逻辑关注点。打个比方，我们来看一下编程语言中的类似概念。许多编程语言，尤其是像 Java 或 C++ 这样的面向对象语言，都带有构造函数或静态方法块。这些语言构造用于初始化字段、验证数据，并在创建类之前设置舞台。并非所有类都需要构造函数，但它们都具备这种能力。

在 Kubernetes 中，这个功能可以通过 init 容器来实现。Init 容器总是在主应用容器之前启动，这意味着它们有自己的生命周期。为了拆分初始化逻辑，甚至可以将工作分配到多个按照清单定义顺序运行的 init 容器中。当然，初始化逻辑可能会失败。如果一个 init 容器产生错误，整个 Pod 将会重新启动，导致所有 init 容器按顺序再次运行。因此，为了防止任何副作用，使 init 容器逻辑幂等是一个好的实践。图 8-1 展示了一个带有两个 init 容器和主应用程序的 Pod。

![ckd2 0801](img/ckd2_0801.png)

###### 图 8-1\. Pod 中 init 容器的顺序和原子生命周期

在过去的几章中，我们探讨了如何在 Pod 中定义一个容器：您只需在 `spec.containers` 下指定其配置即可。对于 init 容器，Kubernetes 提供了一个单独的部分：`spec.initContainers`。Init 容器始终在主应用程序容器之前执行，无论清单中的定义顺序如何。

示例 8-1 中显示的清单定义了一个 init 容器和一个主应用程序容器。大部分情况下，init 容器使用与常规容器相同的属性。但有一个很大的区别。它们不能定义探针，这在 第十四章 中讨论过。init 容器在目录 */usr/shared/app* 中设置了一个配置文件。通过 Volume 共享此目录，以便 Node.js 应用程序在主容器中运行时可以引用它。

##### 示例 8-1\. 定义一个带有 init 容器和主应用容器的 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: business-app
spec:
  initContainers:
  - name: configurer
    image: busybox:1.36.1
    command: ['sh', '-c', 'echo Configuring application... && \
              mkdir -p /usr/shared/app && echo -e "{\"dbConfig\": \
              {\"host\":\"localhost\",\"port\":5432,\"dbName\":\"customers\"}}" \
              > /usr/shared/app/config.json']
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  containers:
  - image: bmuschko/nodejs-read-config:1.0.0
    name: web
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  volumes:
  - name: configdir
    emptyDir: {}
```

启动 Pod 时，您会看到 `get` 命令的状态列也提供有关 init 容器的信息。前缀 `Init:` 表示正在执行 init 容器的过程。冒号后面的状态部分显示了已完成的 init 容器数量与配置的总 init 容器数量：

```
$ kubectl create -f init.yaml
pod/business-app created
$ kubectl get pod business-app
NAME           READY   STATUS    RESTARTS   AGE
business-app   0/1     Init:0/1  0          2s
$ kubectl get pod business-app
NAME           READY   STATUS    RESTARTS   AGE
business-app   1/1     Running   0          8s

```

在执行 init 容器期间可能会出现错误。如果顺序初始化链中任何容器失败，则整个 Pod 将无法启动。您可以始终通过使用 `--container` 命令行选项（或其简短形式 `-c`）来检索 init 容器的日志，如 图 8-2 所示。

![ckd2 0802](img/ckd2_0802.png)

###### 图 8-2\. 针对特定容器

下面的命令显示了 `configurer` init 容器的日志，这相当于我们在 YAML 清单中配置的 `echo` 命令：

```
$ kubectl logs business-app -c configurer
Configuring application...

```

## 旁车模式

初始化容器的生命周期如下：启动、运行逻辑，然后一旦完成工作就终止。初始化容器不打算长时间运行。但有些场景需要不同的使用模式。例如，您可能希望创建一个 Pod，在其中持续运行多个容器。

# Kubernetes 1.29 引入了边车容器。

未来使用 Kubernetes 1.29 或更高版本的考试可能会涵盖正式化的 [边车容器](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)。边车容器是辅助容器，它们将与 Pod 一起启动并在整个 Pod 的生命周期内保持运行。

通常，容器分为两种不同的类别：运行应用程序的容器和为主应用程序提供辅助功能的另一个容器。在 Kubernetes 中，提供辅助功能的容器称为边车。边车容器最常用的功能包括文件同步、日志记录和监视功能。边车通常不是主应用程序的主要流量或 API 的一部分。它们通常是异步操作的，不涉及公共 API。

为了说明边车的行为，考虑以下用例。主应用容器运行一个 Web 服务器——在本例中是 NGINX。一旦启动，Web 服务器会生成两个标准日志文件。文件 */var/log/nginx/access.log* 记录对 Web 服务器端点的请求。另一个文件 */var/log/nginx/error.log* 则记录处理传入请求时的失败。

作为 Pod 功能的一部分，我们希望实现一个监控服务。边车容器定期轮询文件的 *error.log*，检查是否发现任何故障。更具体地说，该服务试图查找日志文件中标有 `[error]` 的故障。如果发现错误，监控服务将对其作出反应。例如，它可以向系统管理员发送通知。我们希望保持功能尽可能简单。监控服务只会将错误消息输出到标准输出。主应用容器和边车容器之间的文件交换通过一个 Volume 实现，如 图 8-3 所示。

![ckd2 0803](img/ckd2_0803.png)

###### 图 8-3\. 边车模式的实际运行情况

在 示例 8-2 中显示的 YAML 文件设置了描述的场景。代码的最棘手部分是长长的 bash 命令。该命令运行一个无限循环。在每次迭代中，我们检查 *error.log* 文件的内容，使用 *grep* 查找错误并可能采取行动。循环每 10 秒执行一次。

##### 示例 8-2\. 典型的边车模式实现

```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
  - name: nginx
    image: nginx:1.25.1
    volumeMounts:
    - name: logs-vol
      mountPath: /var/log/nginx
  - name: sidecar
    image: busybox:1.36.1
    command: ["sh","-c","while true; do if [ \"$(cat /var/log/nginx/error.log \
              | grep 'error')\" != \"\" ]; then echo 'Error discovered!'; fi; \
              sleep 10; done"]
    volumeMounts:
    - name: logs-vol
      mountPath: /var/log/nginx
  volumes:
  - name: logs-vol
    emptyDir: {}
```

在启动 Pod 时，您会注意到总容器数将显示为 2\. 在所有容器都可以启动之后，Pod 会发出 `Running` 状态的信号：

```
$ kubectl create -f sidecar.yaml
pod/webserver created
$ kubectl get pods webserver
NAME        READY   STATUS              RESTARTS   AGE
webserver   0/2     ContainerCreating   0          4s
$ kubectl get pods webserver
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          5s

```

您会发现 *error.log* 最初并不包含任何故障信息。它开始为空文件。通过以下命令，您可以故意引发错误。等待至少 10 秒后，您将在终端上找到预期的消息，您可以使用 `logs` 命令查询该消息：

```
$ kubectl logs webserver -c sidecar
$ kubectl exec webserver -it -c sidecar -- /bin/sh
/ # wget -O- localhost?unknown
Connecting to localhost (127.0.0.1:80)
wget: server returned error: HTTP/1.1 404 Not Found
/ # cat /var/log/nginx/error.log
2020/07/18 17:26:46 [error] 29#29: *2 open() "/usr/share/nginx/html/unknown" \
failed (2: No such file or directory), client: 127.0.0.1, server: localhost, \
request: "GET /unknown HTTP/1.1", host: "localhost"
/ # exit
$ kubectl logs webserver -c sidecar
Error discovered!

```

## 适配器模式

作为应用程序开发者，我们希望专注于实现业务逻辑。例如，在一个为期两周的迭代中，我们被要求添加购物车功能。除了功能需求外，我们还需要考虑暴露管理端点或者制定有意义且正确格式的日志输出等操作方面的内容。很容易陷入将所有方面都包含在应用程序代码中的习惯中，这使得代码更加复杂和难以维护。特别是横切关注点需要在多个应用程序之间复制，并且经常从一个代码库复制粘贴到另一个。

在 Kubernetes 中，我们可以通过在主应用容器之外的另一个容器中运行它们来避免将横切关注点捆绑到应用程序代码中。适配器模式将应用程序产生的输出转换为另一部分系统所需的格式，以使其可消费。图 8-4 展示了适配器模式的一个具体示例。

![ckd2 0804](img/ckd2_0804.png)

###### 图 8-4\. 适配器模式的实际应用

运行主容器的业务应用程序生成带时间戳的信息，例如可用磁盘空间，并将其写入文件 *diskspace.txt*。作为架构的一部分，我们希望从第三方监控应用程序消费该文件。问题在于外部应用程序要求的信息需要排除时间戳。当然，我们可以更改日志格式以避免写入时间戳，但如果我们确实想知道何时写入日志条目呢？这就是适配器模式可以帮助的地方。一个 sidecar 容器执行转换逻辑，将日志条目转换为外部系统所需的格式，而无需更改应用程序逻辑。

YAML 清单中的 示例 8-3 展示了适配器模式的实现可能看起来如何。`app` 容器每五秒生成一个新的日志条目。`transformer` 容器消耗文件内容，移除时间戳，并将其写入新文件。通过一个 Volume，这两个容器都可以访问相同的挂载路径。

##### 示例 8-3\. 一个示例适配器模式的实现

```
apiVersion: v1
kind: Pod
metadata:
  name: adapter
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; \
       sleep 5; done;'
    image: busybox:1.36.1
    name: app
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
  - image: busybox:1.36.1
    name: transformer
    args:
    - /bin/sh
    - -c
    - 'sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" \
       >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < \
       /var/logs/diskspace.txt; sleep 20; done;'
    volumeMounts:
    - name: config-volume
      mountPath: /var/logs
  volumes:
  - name: config-volume
    emptyDir: {}
```

在创建 Pod 后，我们将找到两个正在运行的容器。在进入 `transformer` 容器后，我们应该能够找到原始文件 */var/logs/diskspace.txt*。转换后的数据存在用户主目录中的另一个文件中：

```
$ kubectl create -f adapter.yaml
pod/adapter created
$ kubectl get pods adapter
NAME      READY   STATUS    RESTARTS   AGE
adapter   2/2     Running   0          10s
$ kubectl exec adapter --container=transformer -it -- /bin/sh
/ # cat /var/logs/diskspace.txt
Sun Jul 19 20:28:07 UTC 2020 | 4.0K	/root
Sun Jul 19 20:28:12 UTC 2020 | 4.0K	/root
/ # ls -l
total 40
-rw-r--r--  1  root  root  60 Jul 19 20:28 2020-07-19-20-28-28-transformed.txt
...
/ # cat 2020-07-19-20-28-28-transformed.txt
 4.0K	/root
 4.0K	/root

```

## 大使模式

CKAD 还涵盖了另一个重要的设计模式，即大使模式。大使模式为与外部服务通信提供了一个代理。

许多用例可以证明引入大使模式的合理性。总体目标是隐藏和/或抽象与系统其他部分交互的复杂性。典型的责任包括在请求失败时的重试逻辑、安全问题（如提供认证或授权）以及监控延迟或资源使用情况。图 8-5 展示了这种模式。

![ckd2 0805](img/ckd2_0805.png)

###### 图 8-5\. 大使模式的实际应用

在这个例子中，我们希望为对外部服务的 HTTP(S) 调用实现速率限制功能。例如，速率限制器的要求可能是每 15 分钟最多允许应用程序进行 5 次调用。我们不会将速率限制逻辑紧密耦合到应用程序代码中，而是由大使容器提供。从业务应用程序发出的任何调用都必须通过大使容器进行。示例 8-4 展示了一个基于 Node.js 的速率限制器实现，它向外部服务 [Postman](https://www.postman.com) 发起调用。

##### 示例 8-4\. Node.js HTTP 速率限制器实现

```
const express = require('express');
const app = express();
const rateLimit = require('express-rate-limit');
const https = require('https');

const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message:
    'Too many requests have been made from this IP, please try again after an hour'
});

app.get('/test', rateLimiter, function (req, res) {
  console.log('Received request...');
  var id = req.query.id;
  var url = 'https://postman-echo.com/get?test=' + id;
  console.log("Calling URL %s", url);

  https.get(url, (resp) => {
    let data = '';

    resp.on('data', (chunk) => {
      data += chunk;
    });

    resp.on('end', () => {
      res.send(data);
    });

    }).on("error", (err) => {
      res.send(err.message);
    });
})

var server = app.listen(8081, function () {
  var port = server.address().port
  console.log("Ambassador listening on port %s...", port)
})
```

在 示例 8-5 中展示的对应的 Pod 在不同端口上运行主应用容器和大使容器。对容器 `business-app` 的每次 HTTP 端点调用都会委派给容器 `ambassador` 的 HTTP 端点。需要指出的是，同一 Pod 内运行的容器可以通过 `localhost` 进行通信。不需要额外的网络配置。

##### 示例 8-5\. 一个优秀的大使模式实现

```
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
  - name: business-app
    image: bmuschko/nodejs-business-app:1.0.0
    ports:
    - containerPort: 8080
  - name: ambassador
    image: bmuschko/nodejs-ambassador:1.0.0
    ports:
    - containerPort: 8081
```

让我们来测试功能。首先，我们将创建 Pod，进入运行业务应用程序的容器，并执行一系列 `curl` 命令。前五次调用将被允许访问外部服务。在第六次调用时，由于在给定时间段内达到了速率限制，我们将收到一个错误消息：

```
$ kubectl create -f ambassador.yaml
pod/rate-limiter created
$ kubectl get pods rate-limiter
NAME           READY   STATUS    RESTARTS   AGE
rate-limiter   2/2     Running   0          5s
$ kubectl exec rate-limiter -it -c business-app -- /bin/sh
# curl localhost:8080/test
{"args":{"test":"123"},"headers":{"x-forwarded-proto":"https", \
"x-forwarded-port":"443","host":"postman-echo.com", \
"x-amzn-trace-id":"Root=1-5f177dba-e736991e882d12fcffd23f34"}, \
"url":"https://postman-echo.com/get?test=123"}
...
# curl localhost:8080/test
Too many requests have been made from this IP, please try again after an hour

```

# 总结

真实世界的场景要求在一个 Pod 内部运行多个容器。通过一个初始化容器执行初始化逻辑，可以为主应用容器设置舞台。一旦初始化逻辑完成，该容器将被终止。只有初始化容器顺利运行完其功能后，主应用容器才会启动。

涉及多个容器的其他设计模式包括适配器模式和大使模式。适配器模式有助于“转换”应用程序产生的数据，使其能够被第三方服务使用。大使模式在与外部服务通信时充当应用容器的代理，通过抽象“如何”来实现。

# 考试要点

理解在 Pod 中运行多个容器的必要性。

Pod 可以运行多个容器。你需要了解初始化容器和旁路容器之间的区别及其各自的生命周期。练习在多容器 Pod 中使用命令行选项`--container`或`-c`访问特定容器。

知道如何创建一个初始化容器。

初始化容器在企业 Kubernetes 集群环境中有很高的使用率。理解在各自场景中使用它们的必要性。练习定义一个具有一个或多个初始化容器的 Pod，并在创建 Pod 时观察它们的线性执行。体验在初始化容器发生故障时 Pod 的行为是很重要的。

理解多容器设计模式及其实现方法。

通过实现一个已建立模式的场景，最好理解多容器 Pod。根据你学到的知识，提出你自己的适用案例并创建一个多容器 Pod 来解决它。能够识别旁路模式，并理解它们在实践中为何重要以及如何自己搭建它们是很有帮助的。当你实现自己的旁路容器时，你可能会发现需要复习一下 bash 的知识。

# 示例练习

这些练习的解决方案可以在附录 A 中找到。

1.  为名为`complex-pod`的 Pod 创建一个 YAML 清单。主应用容器名为`app`，使用镜像`nginx:1.25.1`并暴露容器端口 80\. 修改 YAML 清单，使得 Pod 定义一个名为`setup`的初始化容器，使用镜像`busybox:1.36.1`。初始化容器运行命令`wget -O- google.com`。

    从 YAML 清单创建 Pod。

    下载初始化容器的日志。你应该看到`wget`命令的输出。

    打开主应用容器的交互式 Shell，并运行`ls`命令。退出容器。

    强制删除 Pod。

1.  为名为`data-exchange`的 Pod 创建一个 YAML 清单。主应用容器名为`main-app`，使用镜像`busybox:1.36.1`。容器运行的命令是每 30 秒在目录*/var/app/data*中无限循环写入一个新文件，文件名遵循*{counter++}-data.txt*的模式。变量 counter 在每个间隔中递增，初始值为 1。

    修改 YAML 配置文件，添加名为 `sidecar` 的辅助容器。该辅助容器使用镜像 `busybox:1.36.1`，并且在无限循环中每 60 秒统计 `main-app` 容器生成的文件数量。该命令将文件数量输出到标准输出。

    定义一个类型为 `emptyDir` 的卷。为两个容器挂载路径 */var/app/data*。

    创建 Pod。查看 sidecar 容器的日志。

    删除 Pod。
