# Kubernetes Pod 多网卡方案 Multus - 运维派
运维派隶属马哥教育旗下专业运维社区，是国内成立最早的 IT 运维技术社区，欢迎关注公众号：yunweipai  
领取学习更多免费 Linux 云计算、Python、Docker、K8s 教程关注公众号：马哥 linux 运维

在 Kubernetes 中，网络是非常重要的一个领域。Kubernetes 本身不提供网络解决方案，但是提供了 CN I 规范。这些规范被许多 CNI 插件（例如 WeaveNet，Flannel，Calico 等）遵守。这些插件中任何一个都可以在集群上使用和部署以提供网络解决方案。该网络称为集群的默认网络。此默认网络使 Pods 不仅可以在同一节点上而且可以在群集中的各个节点之间相互通信。

随着发展，Kubernetes 缺乏支持 VNF 中多个网络接口的所需功能。传统上，网络功能使用多个网络接口分离控制，管理和控制用户 / 数据的网络平面。他们还用于支持不同的协议，满足不同的调整和配置要求。

为了解决这个需求，英特尔实现了 MULTUS 的 CNI 插件，其中提供了将多个接口添加到 Pod 的功能。这允许 POD 通过不同的接口连接到多个网络，并且每个接口都将使用其自己的 CNI 插件。

下面是 Multus CNI 提供的连接到 Pod 的网络接口的图示。该图显示了具有三个接口的容器：`eth0`，`net0`和`net1`。`eth0`连接 Kubernetes 集群网络以连接 kubernetes 服务器 / 服务（例如 Kubernetes api-server，kubelet 等）。`net0`和`net1`是其他网络附件，并通过使用其他 CNI 插件（例如 vlan / vxlan / ptp）连接到其他网络。

![](http://www.yunweipai.com/wp-content/uploads/2022/06/image-34-780x293.png?v=1656383661)

## **MULTUS 工作原理**

Kubernetes 当前没有提供为 POD 添加额外的接口选项的规定，或支持多个 CNI 插件同时工作的规定，但是它确实提供了一种由 API 服务器扩展受支持的 API 的机制。使用 “自定义资源定义” 可以做到这一点。MULTUS 依赖于 “自定义资源定义” 来存储其他接口和 CNI 插件所需的信息。

![](http://www.yunweipai.com/wp-content/uploads/2022/06/image-35-780x371.png?v=1656383698)

我们首先需要确保将 MULTUS 二进制文件放置在 `/opt/cni/bin` 位置的所有节点上，并在`/etc/cni/net.d`位置创建一个新的配置文件。与 MULTUS 使用的 kubeconfig 文件一起使用。

在`/etc/cni/net.d`中创建的新配置文件基于集群中已经存在的默认网络配置。

在此之后，CRD 用于定义新的种类名称 “NetworkAttachmentDefinition”，以及服务帐户和 MULTUS 的集群角色以及相应的绑定。这个新的集群角色将提供对随 CRD 添加的新 API 组以及默认 API 组中 Pod 资源的访问权限。

然后创建类型为 “NetworkAttachmentDefinition” 的客户资源实例，该实例稍后将在创建具有多个接口的 Pod 时使用。

## 部署示例

在本文中，我们将多次提及两件事：

-   “默认网络” – 这是您的 Pod 到 Pod 网络。这就是集群中 Pod 之间相互通信的方式，以及它们之间的连通性。一般而言，这被称为名为 eth0 的接口。此接口始终连接到您的 Pod，以便它们之间可以相互连接。除此之外，我们还将添加接口。
-   “CRD”    – 自定义资源定义。自定义资源是扩展 Kubernetes API 的一种方式。我们在这里使用这些存储 Multus 可以读取的一些信息。首先，我们使用它们来存储附加到您的 Pod 的每个其他接口的配置。

目前支持 Kubernetes 1.16+ 版本。

## **安装**

我们建议的用于部署 Multus 的快速入门方法是使用 Daemonset（在群集中的每个节点上运行 Pod 的方法）进行部署，该 Pod 会安装 Multus 二进制文件并配置 Multus 以供使用。

首先，克隆此 GitHub 存储库。

    $ git clone https://github.com/intel/multus-cni.git && cd multus-cni

我们将在此存储库中使用带有 kubectl 的 YAML 文件。

    $ cat ./images/multus-daemonset.yml | kubectl apply -f -

### Multus daemonset 完成了那些工作？

-   启动 Multus 守护程序集，这会在每个节点上运行一个 pod，从而在`/opt/cni/bin`中的每个节点上放置一个 Multus 二进制文件
-   按照字母顺序读取`/etc/cni/net.d`中的第一个配置文件，并为 Multus 创建一个新的配置文件，即`/etc/cni/net.d/00-multus.conf`，此配置是自动生成并基于默认网络配置（假定是按字母顺序排列的第一个配置）
-   在每个节点上创建一个`/etc/cni/net.d/multus.d`目录，其中包含用于 Multus 访问 Kubernetes API 的身份验证信息。

## 创建其他接口

我们要做的第一件事是为我们附加到 Pod 的每个其他接口创建配置。我们将通过创建自定义资源来做到这一点。快速入门安装的一部分会创建一个 “CRD” （自定义资源定义，它是我们保留这些自定义资源的位置），我们将在其中存储每个接口的配置。

### CNI 配置

我们将添加的每个配置都是 CNI 配置。如果您不熟悉它们，让我们快速分解它们。这是一个示例 CNI 配置：

    {  
      "cniVersion": "0.3.0",  
      "type": "loopback",  
      "additional": "information"  
    }

CNI 配置是 JSON，我们这里有一个结构，其中包含一些我们感兴趣的东西：

-   `cniVersion`：告诉每个 CNI 插件正在使用哪个版本，如果使用的版本太晚（或太早），则可以提供插件信息。
-   `type`：告诉 CNI 在磁盘上调用哪个二进制文件。每个 CNI 插件都是一个二进制文件。通常，这些二进制文件存储在每个节点上的`/opt/cni/bin`中，并且 CNI 执行此二进制文件。在这种情况下，我们指定了`loopback`二进制文件（它将创建一个`loopback`类型的网络接口）。如果这是您首次安装 Multus，则可能需要验证 “type” 字段中的插件是否确实在`/opt/cni/bin`目录中。
-   `additional`：此字段以此处为例，每个 CNI 插件都可以在 JSON 中指定所需的任何配置参数。这些特定于您在 “type” 字段中调用的二进制文件。

当 CNI 配置更改时，您不需要重新加载或刷新 Kubelets。每次创建和删除 Pod 时都会读取这些内容。因此，如果您更改配置，它将在下一次创建 Pod 时应用。如果现有 Pod 需要新配置，则可能需要重新启动。

### 将配置存储为自定义资源

因此，我们要创建一个附加接口。让我们创建一个 macvlan 接口供 Pod 使用。我们将创建一个自定义资源，该资源定义接口的 CNI 配置。

请注意，在以下命令中有一种：`NetworkAttachmentDefinition`。这是我们配置的名字 - 它是 Kubernetes 的自定义扩展，定义了我们如何将网络连接到 Pod。

其次，注意配置字段。您将看到这是一个 CNI 配置，就像我们前面解释的那样。

最后但非常重要的一点是，在元数据下注意 name 字段 - 在这里我们为该配置指定名称，这是我们告诉 pod 使用此配置的方式。这里的名称是`macvlan-conf`-我们正在为 macvlan 创建配置。

这是创建此示例配置的命令：

    apiVersion: "k8s.cni.cncf.io/v1"  
    kind: NetworkAttachmentDefinition  
    metadata:  
      name: macvlan-conf  
    spec:  
      config: '{  
          "cniVersion": "0.3.0",  
          "type": "macvlan",  
          "master": "eth0",  
          "mode": "bridge",  
          "ipam": {  
            "type": "host-local",  
            "subnet": "192.168.1.0/24",  
            "rangeStart": "192.168.1.200",  
            "rangeEnd": "192.168.1.216",  
            "routes": [  
              { "dst": "0.0.0.0/0" }  
            ],  
            "gateway": "192.168.1.1"  
          }  
        }'

> 本示例使用 eth0 作为主参数，此主参数应与集群中主机上的接口名称匹配。

您可以查看使用 kubectl 创建的配置，方法如下：

    $ kubectl get network-attachment-definitions

您可以通过描述它们来获得更多详细信息：

    $ kubectl describe network-attachment-definitions macvlan-conf

### 创建一个附加附加接口的 Pod

我们将创建一个 pod。就像您之前可能创建的任何 pod 一样，它看起来都很熟悉，但是，我们将有一个特殊的注释字段 - 在这种情况下，我们将有一个名为`k8s.v1.cni.cncf.io/networks`的注释。如上创建的，该字段以逗号分隔的列表列出了 NetworkAttachmentDefinitions 的名称。请注意，在下面的命令中，我们具有 `k8s.v1.cni.cncf.io/networks` 的注释：`macvlan-conf`其中`macvlan-conf`是我们在创建配置时使用的名称。

让我们继续使用以下命令创建一个 pod：

    apiVersion: v1  
    kind: Pod  
    metadata:  
      name: samplepod  
      annotations:  
        k8s.v1.cni.cncf.io/networks: macvlan-conf  
    spec:  
      containers:  
      - name: samplepod  
        command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]  
        image: alpine

您现在可以检查 Pod 并查看连接了哪些接口，如下所示：

    $ kubectl exec -it samplepod -- ip a

您应该看到，有 3 个接口：

-   `lo`环回接口
-   `eth0`我们的默认网络
-   `net1`是我们使用 macvlan 配置创建的新接口

### 网络状态 Annotations

为了确认，请使用`kubectl describe pod pod samplepod`，然后会有一个注释部分，类似于以下内容：

    Annotations:        k8s.v1.cni.cncf.io/networks: macvlan-conf  
                        k8s.v1.cni.cncf.io/networks-status:  
                          [{  
                              "name": "cbr0",  
                              "ips": [  
                                  "10.244.1.73"  
                              ],  
                              "default": **true**,  
                              "dns": {}  
                          },{  
                              "name": "macvlan-conf",  
                              "interface": "net1",  
                              "ips": [  
                                  "192.168.1.205"  
                              ],  
                              "mac": "86:1d:96:ff:55:0d",  
                              "dns": {}  
                          }]

该元数据告诉我们，我们有两个成功运行的 CNI 插件。

### 如果我想要更多接口怎么办？

您可以通过创建更多的自定义资源，然后在 pod 的注释中引用它们，来向 pod 添加更多接口。您还可以重复使用配置，例如，要将两个 macvlan 接口附加到 Pod，可以创建如下 Pod：

    apiVersion: v1  
    kind: Pod  
    metadata:  
      name: samplepod  
      annotations:  
        k8s.v1.cni.cncf.io/networks: macvlan-conf,macvlan-conf  
    spec:  
      containers:  
      - name: samplepod  
        command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]  
        image: alpine

请注意，注释现在读取为`k8s.v1.cni.cncf.io/networks：macvlan-conf，macvlan-conf`。如果我们有两次使用相同的配置，并用逗号分隔。

_链接：[https://houmin.cc/posts/53d9ea76/](https://houmin.cc/posts/53d9ea76/)_（侵删）

![](http://www.yunweipai.com/wp-content/uploads/2022/06/%E4%BA%91%E5%8E%9F%E7%94%9F%E7%AB%96%E7%89%88%E6%B5%B7%E6%8A%A5%EF%BC%886%E6%9C%8830%E6%97%A5%E8%BF%87%E6%9C%9F%EF%BC%89.jpg)

本文链接：[http://www.yunweipai.com/41890.html](http://www.yunweipai.com/41890.html) 
 [http://www.yunweipai.com/41890.html](http://www.yunweipai.com/41890.html)
