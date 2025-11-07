## 附录 C. Vagrant

在本书的各个部分，我们使用虚拟机来演示 Docker 需要完整机器表示或甚至多个虚拟机编排的技术。Vagrant 提供了一种简单的方法，可以从命令行启动、配置和管理虚拟机，并且它在多个平台上都可用。

## 设置

访问 [`www.vagrantup.com`](https://www.vagrantup.com) 并遵循那里的说明来设置。

## 图形用户界面

当运行 `vagrant up` 以启动虚拟机时，Vagrant 会读取名为 Vagrantfile 的本地文件以确定设置。

你可以在你的 `provider` 部分创建或更改的一个有用设置是 `gui`：

```
v.gui = true
```

例如，如果你的提供者是 VirtualBox，一个典型的配置部分可能看起来像这样：

```
Vagrant.configure(2) do |config|
  config.vm.box = "hashicorp/precise64"

  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
    v.gui = false
  end
end
```

在运行 `vagrant up` 以启动虚拟机之前，你可以将 `v.gui` 行的 `false` 设置更改为 `true`（或者如果它还没有添加，则添加它）以获得正在运行的虚拟机的图形用户界面。


##### 小贴士

Vagrant 中的 *provider* 是提供虚拟机环境的程序的名称。对于大多数用户来说，这将是 `virtualbox`，但它也可能是 `libvirt`、`openstack` 或 `vmware_fusion`（以及其他）。


## 内存

Vagrant 使用虚拟机来创建其环境，这些环境可能会非常消耗内存。如果你正在运行一个由每个虚拟机占用 2 GB 内存的三节点集群，你的机器将需要 6 GB 的可用内存。如果你的机器运行困难，这种内存不足很可能是原因——唯一的解决方案是停止任何非必要的虚拟机或购买更多内存。能够避免这种情况是 Docker 比虚拟机更强大的原因之一——你不需要预先为容器分配资源——它们只需消耗它们需要的。
