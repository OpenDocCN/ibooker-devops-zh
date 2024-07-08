# 第八章：卷和配置数据

在 Kubernetes 中，*volume* 是一个对所有在 pod 中运行的容器可访问的目录，还保证数据跨单个容器重启时保持不变。

我们可以区分几种类型的卷：

+   *节点本地* 临时卷，例如 `emptyDir`

+   通用*网络*卷，例如 `nfs` 或 `cephfs`

+   *云服务提供商特定* 卷，例如 `AWS EBS` 或 `AWS EFS`

+   *特殊用途*卷，例如 `secret` 或 `configMap`

你选择哪种卷类型完全取决于你的用例。例如，对于临时的临时空间，`emptyDir` 可能足够了，但是当你需要确保数据在节点故障时能够存活时，你需要考虑更具弹性的替代方案或特定于云提供商的解决方案。

# 8.1 通过本地卷在容器之间交换数据

## 问题

你有两个或更多的容器在一个 pod 中运行，并希望能够通过文件系统操作交换数据。

## 解决方案

使用类型为 `emptyDir` 的本地卷。

起始点是以下 pod 清单 *exchangedata.yaml*，其中有两个容器（`c1` 和 `c2`），它们每个都将本地卷 `xchange` 挂载到它们的文件系统中，使用不同的挂载点：

```
apiVersion: v1
kind: Pod
metadata:
  name: sharevol
spec:
  containers:
  - name: c1
    image: ubuntu:20.04
    command:
      - "bin/bash"
      - "-c"
      - "sleep 10000"
    volumeMounts:
      - name: xchange
        mountPath: "/tmp/xchange"
  - name: c2
    image: ubuntu:20.04
    command:
      - "bin/bash"
      - "-c"
      - "sleep 10000"
    volumeMounts:
      - name: xchange
        mountPath: "/tmp/data"
  volumes:
  - name: xchange
    emptyDir: {}
```

现在你可以启动 pod，`exec` 进入它，从一个容器中创建数据，然后从另一个容器中读取它：

```
$ kubectl apply -f exchangedata.yaml
pod/sharevol created

$ kubectl exec sharevol -c c1 -i -t -- bash
[root@sharevol /]# mount | grep xchange
/dev/vda1 on /tmp/xchange type ext4 (rw,relatime)
[root@sharevol /]# echo 'some data' > /tmp/xchange/data
[root@sharevol /]# exit

$ kubectl exec sharevol -c c2 -i -t -- bash
[root@sharevol /]# mount | grep /tmp/data
/dev/vda1 on /tmp/data type ext4 (rw,relatime)
[root@sharevol /]# cat /tmp/data/data
some data
[root@sharevol /]# exit

```

## 讨论

本地卷由运行 pod 及其容器的节点支持。如果节点宕机或需要对其进行维护（参见 Recipe 12.9），那么本地卷就会丢失，并且所有数据也会丢失。

有些用例中，本地卷是可以的——例如一些临时空间或者从其他地方（例如 S3 存储桶）获取规范状态时——但一般来说，你应该使用持久卷或由网络存储支持的卷（参见 Recipe 8.4）。

## 参见

+   Kubernetes 关于 [volumes](https://oreil.ly/82P1u) 的文档

# 8.2 使用秘密向 pod 传递 API 访问密钥

## 问题

作为管理员，你希望以安全的方式向你的开发者提供 API 访问密钥，即在 Kubernetes 清单中不明文共享它。

## 解决方案

使用类型为 [`secret`](https://oreil.ly/bX6ER) 的本地卷。

假设你想要让你的开发者能够访问一个通过口令 `open sesame` 访问的外部服务。

首先，创建一个名为 *passphrase* 的文本文件，其中包含口令：

```
$ echo -n "open sesame" > ./passphrase

```

接下来，使用 *passphrase* 文件创建 [secret](https://oreil.ly/cCddB)：

```
$ kubectl create secret generic pp --from-file=./passphrase
secret/pp created

$ kubectl describe secrets/pp
Name:           pp
Namespace:      default
Labels:         <none>
Annotations:    <none>

Type:   Opaque

Data
====
passphrase:     11 bytes

```

从管理员角度来看，你现在已经准备就绪，是开发者消费秘密的时候了。所以让我们换个角色，假设你是一个开发者，想要在 pod 内部使用口令。

例如，你可以将其作为一个卷挂载到你的 pod 中，然后像普通文件一样读取它。创建并保存以下名为 *ppconsumer.yaml* 的清单：

```
apiVersion: v1
kind: Pod
metadata:
  name: ppconsumer
spec:
  containers:
  - name: shell
    image: busybox:1.36
    command:
      - "sh"
      - "-c"
      - "mount | grep access  && sleep 3600"
    volumeMounts:
      - name: passphrase
        mountPath: "/tmp/access"
        readOnly: true
  volumes:
  - name: passphrase
    secret:
      secretName: pp
```

现在启动 Pod 并查看其日志，您期望看到`ppconsumer`密钥文件挂载为*/tmp/access/passphrase*：

```
$ kubectl apply -f ppconsumer.yaml
pod/ppconsumer created

$ kubectl logs ppconsumer
tmpfs on /tmp/access type tmpfs (ro,relatime,size=7937656k)

```

要从运行中的容器中访问密码，只需读取*/tmp/access*中的*passphrase*文件，如下所示：

```
$ kubectl exec ppconsumer -i -t -- sh

/ # cat /tmp/access/passphrase
open sesame
/ # exit

```

## 讨论

秘密存在于命名空间的上下文中，因此在设置它们和/或使用它们时需要考虑到这一点。

您可以通过以下方法之一访问运行在 Pod 中的容器的秘密：

+   一个卷（如解决方案中所示，内容存储在`tmpfs`卷中）

+   [将秘密作为环境变量使用](https://oreil.ly/Edsr5)

此外，请注意秘密的大小限制为 1 MiB。

###### 小贴士

`kubectl create secret`处理三种类型的秘密，根据您的用例，您可能需要选择不同的秘密类型：

+   `docker-registry`类型用于与 Docker 注册表一起使用。

+   `generic`类型是我们在解决方案中使用的；它从本地文件、目录或文字值创建秘密（您需要自行进行 base64 编码）。

+   使用`tls`，例如，您可以创建一个安全的 SSL 证书用于入口。

`kubectl describe`不会以纯文本显示秘密内容。这避免了“偷看”密码。但是，由于它只是 base64 编码而不是加密，您可以轻松地手动解码它：

```
$ kubectl get secret pp -o yaml | \
    grep passphrase | \
    cut -d":" -f 2 | \
    awk '{$1=$1};1' | \
    base64 --decode
open sesame

```

在此命令中，第一行检索秘密的 YAML 表示，第二行使用`grep`提取`passphrase: b3BlbiBzZXNhbWU=`行（请注意这里的前导空格）。然后，`cut`提取出密码的内容，`awk`命令去掉前导空格。最后，`base64`命令将其转换回原始数据。

###### 小贴士

您可以使用`--encryption-provider-config`选项在启动`kube-apiserver`时选择加密秘密。

## 另请参阅

+   Kubernetes 关于[秘密](https://oreil.ly/cCddB)的文档

+   Kubernetes 关于[加密静态数据](https://oreil.ly/kAmrN)的文档

# 8.3 提供配置数据给应用程序

## 问题

您希望为应用程序提供配置数据，而不将其存储在容器映像中或硬编码到 Pod 规范中。

## 解决方案

使用配置映射。这些是 Kubernetes 的一流资源，您可以通过环境变量或文件向 Pod 提供配置数据。

假设您想创建一个带有键`siseversion`和值`0.9`的配置。只需这样简单：

```
$ kubectl create configmap nginxconfig \
    --from-literal=nginxgreeting="hello from nginx"
configmap/nginxconfig created

```

现在您可以在部署中使用配置映射，比如说，在一个具有以下内容的清单文件中：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25.2
    env:
    - name: NGINX_GREETING
      valueFrom:
        configMapKeyRef:
          name: nginxconfig
          key: nginxgreeting
```

将此 YAML 清单保存为*nginxpod.yaml*，然后使用`kubectl`创建 Pod：

```
$ kubectl apply -f nginxpod.yaml
pod/nginx created

```

您可以使用以下命令列出 Pod 的容器环境变量：

```
$ kubectl exec nginx -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx
NGINX_GREETING=hello from nginx
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
...

```

## 讨论

我们刚刚展示了如何将配置作为环境变量传递。但是，您也可以将其作为文件挂载到 Pod 中，使用一个卷。

假设您有以下配置文件 *example.cfg*：

```
debug: true
home: ~/abc
```

您可以创建一个包含配置文件的配置映射，如下所示：

```
$ kubectl create configmap configfile --from-file=example.cfg
configmap/configfile created

```

现在，您可以像使用任何其他卷一样使用配置映射。以下是名为 `oreilly` 的 Pod 的清单文件；它使用 `busybox` 镜像并休眠 3,600 秒。在 `volumes` 部分，有一个名为 `oreilly` 的卷，它使用我们刚刚创建的配置映射 `configfile`。然后，在容器内部的路径 `/oreilly` 处挂载此卷。因此，文件将在 Pod 内可访问：

```
apiVersion: v1
kind: Pod
metadata:
  name: oreilly
spec:
  containers:
  - image: busybox:1.36
    command:
      - sleep
      - "3600"
    volumeMounts:
    - mountPath: /oreilly
      name: oreilly
    name: busybox
  volumes:
  - name: oreilly
    configMap:
      name: configfile
```

创建完 Pod 后，您可以验证 *example.cfg* 文件确实存在其中：

```
$ kubectl exec -ti oreilly -- ls -l oreilly
total 0
lrwxrwxrwx   1 root   root   18 Mar 31 09:39 example.cfg -> ..data/example.cfg

$ kubectl exec -ti oreilly -- cat oreilly/example.cfg
debug: true
home: ~/abc

```

关于如何从文件创建配置映射的完整示例，请参见 Recipe 11.7。

## 另请参阅

+   [“配置 Pod 使用 ConfigMap”](https://oreil.ly/R1FgU) 在 Kubernetes 文档中

# 8.4 在 Minikube 中使用持久卷

## 问题

您不希望在容器使用的磁盘上丢失数据，即希望确保它在托管 Pod 重新启动后能够存活。

## 解决方案

使用持久卷（PV）。在 Minikube 的情况下，您可以创建一个 `hostPath` 类型的 PV，并像普通卷一样将其挂载到容器的文件系统中。

首先，在名为 *hostpath-pv.yaml* 的清单中定义 PV `hostpathpv`：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpathpv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/tmp/pvdata"
```

但在创建 PV 之前，您需要准备节点上的目录 */tmp/pvdata*，即 Minikube 实例本身。您可以使用 `minikube ssh` 进入运行 Kubernetes 集群的节点：

```
$ minikube ssh

$ mkdir /tmp/pvdata && \
    echo 'I am content served from a delicious persistent volume' > \
    /tmp/pvdata/index.html

$ cat /tmp/pvdata/index.html
I am content served from a delicious persistent volume

$ exit

```

现在，您已经在节点上准备好了目录，可以从清单文件 *hostpath-pv.yaml* 创建 PV：

```
$ kubectl apply -f hostpath-pv.yaml
persistentvolume/hostpathpv created

$ kubectl get pv
NAME        CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      ...   ...   ...
hostpathpv  1Gi        RWO           Retain          Available   ...   ...   ...

$ kubectl describe pv/hostpathpv
Name:            hostpathpv
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual
Status:          Available
Claim:
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/pvdata
    HostPathType:
Events:            <none>

```

到目前为止，您将在管理员角色中执行这些步骤。您将定义 PV 并使其在 Kubernetes 集群中对开发者可用。

现在从开发者的角度来看，您可以在 Pod 中使用 PV。这是通过*持久卷声明*（PVC）完成的，因为您确实声明了一个 PV，满足某些特征，如大小或存储类。

创建一个名为 *pvc.yaml* 的清单文件，定义一个 PVC，请求 200 MB 的空间：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
```

接下来，启动 PVC 并验证其状态：

```
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/mypvc created

$ kubectl get pv
NAME        CAPACITY  ACCESSMODES  ...  STATUS  CLAIM          STORAGECLASS
hostpathpv  1Gi       RWO          ...  Bound   default/mypvc  manual

```

注意，PV `hostpathpv` 的状态已从 `Available` 变为 `Bound`。

最后，是时候在容器中消耗 PV 中的数据了，这次是通过将其挂载到文件系统中的部署完成的。因此，请创建一个名为 *nginx-using-pv.yaml* 的文件，并包含以下内容：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-pv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: webserver
        image: nginx:1.25.2
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: webservercontent
      volumes:
      - name: webservercontent
        persistentVolumeClaim:
          claimName: mypvc
```

并启动部署，如下所示：

```
$ kubectl apply -f nginx-using-pv.yaml
deployment.apps/nginx-with-pv created

$ kubectl get pvc
NAME   STATUS  VOLUME      CAPACITY  ACCESSMODES  STORAGECLASS  AGE
mypvc  Bound   hostpathpv  1Gi       RWO          manual        12m

```

正如您所看到的，PV 正通过您之前创建的 PVC 使用中。

要验证数据确实已到达，您现在可以创建一个服务（参见 Recipe 5.1）以及一个 `Ingress` 对象（参见 Recipe 5.5），然后像这样访问它：

```
$ curl -k -s https://192.168.99.100/web
I am content served from a delicious persistent volume

```

干得好！您（作为管理员）已经提供了持久卷，并（作为开发者）通过持久卷声明对其进行了声明，并通过将其挂载到容器文件系统中的 Pod 中使用。

## 讨论

在解决方案中，我们使用了 `hostPath` 类型的持久卷。在生产环境中，您不希望使用这个，而是要求您的集群管理员提供由 NFS 或 Amazon Elastic Block Store (EBS) 卷支持的网络化卷，以确保您的数据保持和在单节点故障时能够存活。

###### 注意

请记住，PV 是集群范围的资源；即它们不是命名空间的。但是，PVC 是命名空间的。您可以使用命名空间 PVC 从特定命名空间声明 PV。

## 参见

+   Kubernetes [持久卷文档](https://oreil.ly/IMCId)

+   [“配置 Pod 使用持久卷进行存储”](https://oreil.ly/sNDkp) 在 Kubernetes 文档中

# 8.5 在 Minikube 上理解数据持久性

## 问题

您希望使用 Minikube 在 Kubernetes 中部署一个有状态的应用程序。具体来说，您想部署一个 MySQL 数据库。

## 解决方案

在您的 pod 定义和/或数据库模板中使用 `PersistentVolumeClaim` 对象（参见 Recipe 8.4）。

首先，您需要请求特定数量的存储空间。以下的 *data.yaml* 清单请求了 1 GB 的存储空间：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

在 Minikube 上，创建此 PVC 并立即查看如何创建与此声明匹配的持久卷：

```
$ kubectl apply -f data.yaml
persistentvolumeclaim/data created

$ kubectl get pvc
NAME  STATUS  VOLUME                                    CAPACITY ...  ...  ...
data  Bound   pvc-da58c85c-e29a-11e7-ac0b-080027fcc0e7  1Gi      ...  ...  ...

$ kubectl get pv
NAME                                      CAPACITY  ...  ...  ...  ...  ...
pvc-da58c85c-e29a-11e7-ac0b-080027fcc0e7  1Gi       ...  ...  ...  ...  ...

```

您现在可以在您的 pod 中使用此声明。在 pod 清单的 `volumes` 部分，通过名称定义一个 PVC 类型的卷，并引用您刚刚创建的 PVC。

在 `volumeMounts` 字段中，您将在容器内的特定路径挂载此卷。对于 MySQL，您将其挂载在 `/var/lib/mysql` 下：

```
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
  - image: mysql:8.1.0
    name: db
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: data
    env:
      - name: MYSQL_ROOT_PASSWORD
        value: root
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
```

## 讨论

Minikube 默认配置了一个默认存储类别，定义了默认的持久卷供应者。这意味着当创建持久卷声明时，Kubernetes 将动态创建匹配的持久卷以填充该声明。

这就是解决方案中发生的事情。当您创建名为 `data` 的持久卷声明时，Kubernetes 自动创建了一个持久卷来匹配该声明。如果您稍微深入查看 Minikube 上的默认存储类别，您会看到供应商类型：

```
$ kubectl get storageclass
NAME                 PROVISIONER                ...
standard (default)   k8s.io/minikube-hostpath   ...

$ kubectl get storageclass standard -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
...
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete

```

此特定存储类别使用一个创建 `hostPath` 类型的存储供应者。您可以通过查看先前创建的匹配声明的 PV 的清单来查看这一点：

```
$ kubectl get pv
NAME                                       CAPACITY   ... CLAIM         ...
pvc-da58c85c-e29a-11e7-ac0b-080027fcc0e7   1Gi        ... default/data  ...

$ kubectl get pv pvc-da58c85c-e29a-11e7-ac0b-080027fcc0e7 -o yaml
apiVersion: v1
kind: PersistentVolume
...
  hostPath:
    path: /tmp/hostpath-provisioner/default/data
    type: ""
...

```

要验证所创建的主机卷是否包含数据库 `data`，您可以连接到 Minikube 并列出目录中的文件：

```
$ minikube ssh

$ ls -l /tmp/hostpath-provisioner/default/data
total 99688
...
drwxr-x--- 2 999 docker     4096 Mar 31 11:11  mysql
-rw-r----- 1 999 docker 31457280 Mar 31 11:11  mysql.ibd
lrwxrwxrwx 1 999 docker       27 Mar 31 11:11  mysql.sock -> /var/run/mysqld/...
drwxr-x--- 2 999 docker     4096 Mar 31 11:11  performance_schema
-rw------- 1 999 docker     1680 Mar 31 11:11  private_key.pem
-rw-r--r-- 1 999 docker      452 Mar 31 11:11  public_key.pem
...

```

确实，现在您拥有数据持久性。如果 pod 崩溃（或者您删除它），您的数据仍将可用。

通常，存储类别允许集群管理员定义他们可能提供的各种存储类型。对于开发人员来说，这抽象了存储类型，让他们可以使用 PVC 而不必担心存储提供者本身。

## 参见

+   [Kubernetes 持久卷声明文档](https://oreil.ly/8CRZI)

+   [Kubernetes 存储类文档](https://oreil.ly/32-fw)

# 8.6 在版本控制中存储加密的秘密

## 问题

您希望将所有 Kubernetes 清单存储在版本控制中并安全共享它们（甚至是公开的），包括秘密。

## 解决方案

使用[sealed-secrets](https://oreil.ly/r-83j)。Sealed-secrets 是一个 Kubernetes 控制器，用于解密单向加密的秘密并在集群中创建`Secret`对象（参见 Recipe 8.2）。

要开始，请从[发布页面](https://oreil.ly/UgMpf)安装`v0.23.1`版本的 sealed-secrets 控制器：

```
$ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/
releases/download/v0.23.1/controller.yaml

```

结果将是您在`kube-system`命名空间中运行一个新的自定义资源和一个新的 pod：

```
$ kubectl get customresourcedefinitions
NAME                        CREATED AT
sealedsecrets.bitnami.com   2023-01-18T09:23:33Z

$ kubectl get pods -n kube-system  -l name=sealed-secrets-controller
NAME                                         READY   STATUS    RESTARTS   AGE
sealed-secrets-controller-7ff6f47d47-dd76s   1/1     Running   0          2m22s

```

接下来，从[发布页面](https://oreil.ly/UgMpf)下载相应版本的`kubeseal`二进制文件。这个工具将允许您加密您的秘密。

例如，在 macOS（amd64）上，执行以下操作：

```
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/
v0.23.1/kubeseal-0.23.1-darwin-amd64.tar.gz

$ tar xf kubeseal-0.23.1-darwin-amd64.tar.gz

$ sudo install -m 755 kubeseal /usr/local/bin/kubeseal

$ kubeseal --version
kubeseal version: 0.23.1

```

您现在可以开始使用密封的秘密了。首先，生成一个通用的秘密清单：

```
$ kubectl create secret generic oreilly --from-literal=password=root -o json \
    --dry-run=client > secret.json

$ cat secret.json
{
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
        "name": "oreilly",
        "creationTimestamp": null
    },
    "data": {
        "password": "cm9vdA=="
    }
}

```

然后使用`kubeseal`命令生成新的自定义`SealedSecret`对象：

```
$ kubeseal < secret.json > sealedsecret.json

$ cat sealedsecret.json
{
  "kind": "SealedSecret",
  "apiVersion": "bitnami.com/v1alpha1",
  "metadata": {
    "name": "oreilly",
    "namespace": "default",
    "creationTimestamp": null
  },
  "spec": {
    "template": {
      "metadata": {
        "name": "oreilly",
        "namespace": "default",
        "creationTimestamp": null
      }
    },
    "encryptedData": {
      "password": "AgCyN4kBwl/eLt7aaaCDDNlFDp5s93QaQZZ/mm5BJ6SK1WoKyZ45hz..."
    }
  }
}

```

使用以下命令创建`SealedSecret`对象：

```
$ kubectl apply -f sealedsecret.json
sealedsecret.bitnami.com/oreilly created

```

您现在可以安全地将*sealedsecret.json*存储在版本控制中。

## 讨论

创建`SealedSecret`对象后，控制器将检测到它，解密它，并生成相应的秘密。

您的敏感信息被加密到一个`SealedSecret`对象中，这是一个自定义资源（参见 Recipe 15.4）。`SealedSecret`安全地存储在版本控制下并共享，甚至是公开的。一旦在 Kubernetes API 服务器上创建了`SealedSecret`，只有存储在密封秘密控制器中的私钥才能解密它并创建相应的`Secret`对象（仅为 base64 编码）。没有其他人，包括原作者，可以从`Se⁠al⁠ed​Se⁠cr⁠et`中解密原始`Secret`。

虽然用户无法从`SealedSecret`中解密原始`Secret`，但他们可能能够从集群中访问未封存的`Secret`。您应该配置 RBAC 以禁止低权限用户从他们有限制访问权限的命名空间中读取`Secret`对象。

您可以使用以下命令列出当前命名空间中的`SealedSecret`对象：

```
$ kubectl get sealedsecret
NAME      AGE
oreilly   14s

```

## 参见

+   [GitHub 上的密封秘密项目](https://oreil.ly/SKVWq)

+   Angus Lees 的文章[“密封的秘密：在到达 Kubernetes 之前保护您的密码”](https://oreil.ly/Ie3nB)
