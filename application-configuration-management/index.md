# 应用配置管理


## 需求来源
### 背景问题
首先一起来看一下需求来源。大家应该都有过这样的经验，用一个容器镜像来启动一个 container。要启动这个容器，其实有很多需要配套的问题待解决：

* 第一，比如说一些可变的配置。因为我们不可能把一些可变的配置写到镜像里面，当这个配置需要变化的时候，可能需要我们重新编译一次镜像，这个肯定是不能接受的；
* 第二就是一些敏感信息的存储和使用。比如说应用需要使用一些密码，或者用一些 token；
* 第三就是我们容器要访问集群自身。比如我要访问 kube-apiserver，那么本身就有一个身份认证的问题；
* 第四就是容器在节点上运行之后，它的资源需求；
* 第五个就是容器在节点上，它们是共享内核的，那么它的一个安全管控怎么办？

最后一点，容器启动之前的一个前置条件检验。比如说，一个容器启动之前，可能要确认一下 DNS 服务是不是好用？又或者确认一下网络是不是联通的？那么这些其实就是一些前置的校验。

### Pod 的配置管理

在 Kubernetes 里面，它是怎么做这些配置管理的呢？如下表所示：

|                    配置                     |   说明   |
| :-----------------------------------------: | :------: |
|                  ConfigMap                  | 可变配置 |
|                   Secret                    | 敏感信息 |
|               ServiceAccount                | 身份认证 |
| Spec.Containers[].Resources.limits/requests | 资源配置 |
| Spec.Containers[].Resources.SecrityContext  | 安全管控 |
|             Spec.InitContainers             | 前置校验 |

## ConfigMap

### ConfigMap 介绍

ConfigMap 它是用来做什么的、以及它带来的一个好处。它其实主要是管理一些可变配置信息，比如说我们应用的一些配置文件，或者说它里面的一些环境变量，或者一些命令行参数。

它的好处在于它可以让一些可变配置和容器镜像进行解耦，这样也保证了容器的可移植性。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  lables:
    app: flannel
    tier: node
  name: kube-flannnel-cfg
  namespace: kube-system
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": ""flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "172.27.0.0/16",
      "Backend": {
        "Type": "vxlan"
        }
      }
```

这是 ConfigMap 本身的一个定义，它包括两个部分：一个是 ConfigMap 元信息，我们关注 name 和 namespace 这两个信息。接下来这个 data 里面，可以看到它管理了两个配置文件。它的结构其实是这样的：从名字看ConfigMap中包含Map单词，Map 其实就是 key:value，key 是一个文件名，value 是这个文件的内容。

### ConfigMap 创建

推荐用 kubectl 这个命令来创建，它带的参数主要有两个：一个是指定 name，第二个是 DATA。其中 DATA 可以通过指定文件或者指定目录，以及直接指定键值对。

```
kubectl create configmap [NAME] [DATA]
```

指定文件的话，文件名就是 Map 中的 key，文件内容就是 Map 中的 value。然后指定键值对就是指定数据键值对，即：key:value 形式，直接映射到 Map 的key:value。

### ConfigMap 使用
创建完了之后，应该怎么使用呢？

* 第一种是环境变量。环境变量的话通过 valueFrom，然后 ConfigMapKeyRef 这个字段，下面的 name 是指定 ConfigMap 名，key 是 ConfigMap.data 里面的 key。这样的话，在 busybox 容器启动后容器中执行 env 将看到一个 SPECIAL_LEVEL_KEY 环境变量；
* 第二个是命令行参数。命令行参数其实是第一行的环境变量直接拿到 cmd 这个字段里面来用；
* 最后一个是通过 volume 挂载的方式直接挂到容器的某一个目录下面去。上面的例子是把 special-config 这个 ConfigMap 里面的内容挂到容器里面的 /etc/config 目录下，这个也是使用的一种方式。

### ConfigMap 注意要点
现在对 ConfigMap 的使用做一个总结，以及它的一些注意点，注意点一共列了以下五条：

1. ConfigMap 文件的大小。虽然说 ConfigMap 文件没有大小限制，但是在 ETCD 里面，数据的写入是有大小限制的，现在是限制在 1MB 以内；
2. 第二个注意点是 pod 引入 ConfigMap 的时候，必须是相同的 Namespace 中的 ConfigMap，前面其实可以看到，ConfigMap.metadata 里面是有 namespace 字段的；
3. 第三个是 pod 引用的 ConfigMap。假如这个 ConfigMap 不存在，那么这个 pod 是无法创建成功的，其实这也表示在创建 pod 前，必须先把要引用的 ConfigMap 创建好；
4. 第四点就是使用 envFrom 的方式。把 ConfigMap 里面所有的信息导入成环境变量时，如果 ConfigMap 里有些 key 是无效的，比如 key 的名字里面带有数字，那么这个环境变量其实是不会注入容器的，它会被忽略。但是这个 pod 本身是可以创建的。这个和第三点是不一样的方式，是 ConfigMap 文件存在基础上，整体导入成环境变量的一种形式；
5. 最后一点是：什么样的 pod 才能使用 ConfigMap？这里只有通过 K8s api 创建的 pod 才能使用 ConfigMap，比如说通过用命令行 kubectl 来创建的 pod，肯定是可以使用 ConfigMap 的，但其他方式创建的 pod，比如说 kubelet 通过 manifest 创建的 static pod，它是不能使用 ConfigMap 的。

## Secret
### Secret 介绍
Secret 是一个主要用来存储密码 token 等一些敏感信息的资源对象。其中，敏感信息是采用 base-64 编码保存起来的。

Secret 类型种类比较多，下面列了常用的四种类型：

* 第一种是 Opaque，它是普通的 Secret 文件；
* 第二种是 service-account-token，是用于 service-account 身份认证用的 Secret；
* 第三种是 dockerconfigjson，这是拉取私有仓库镜像的用的一种 Secret；
* 第四种是 bootstrap.token，是用于节点接入集群校验用的 Secret。

### Secret 创建
Secret 有两种创建方式：

* 系统创建：比如 K8s 为每一个 namespace 的默认用户（default ServiceAccount）创建 Secret；
* 用户手动创建：手动创建命令，推荐 kubectl 这个命令行工具，它相对 ConfigMap 会多一个 type 参数。其中 data 也是一样，它也是可以指定文件和键值对的。type 的话，要是你不指定的话，默认是 Opaque 类型。

### Secret 使用
创建完 Secret 之后，再来看一下如何使用它。它主要是被 pod 来使用，一般是通过 volume 形式挂载到容器里指定的目录，然后容器里的业务进程再到目录下读取 Secret 来进行使用。另外在需要访问私有镜像仓库时，也是通过引用 Secret 来实现。

挂载到用户指定目录的方式：

* 第一种方式：用户直接指定
* 第二种方式：系统自动生成，过程中会生成两个文件，一个是 ca.crt，一个是 token。

### Secret 使用注意要点

1. Secret 的文件大小限制。这个跟 ConfigMap 一样，也是 1MB。
2. Secret 采用了 base-64 编码，但是它跟明文也没有太大区别。所以说，如果有一些机密信息要用 Secret 来存储的话，还是要很慎重考虑。也就是说谁会来访问你这个集群，谁会来用你这个 Secret，还是要慎重考虑，因为它如果能够访问这个集群，就能拿到这个 Secret。如果是对 Secret 敏感信息要求很高，对加密这块有很强的需求，推荐可以使用 Kubernetes 和开源的 vault做一个解决方案，来解决敏感信息的加密和权限管理。
3. Secret 读取的最佳实践，建议不要用 list/watch，如果用 list/watch 操作的话，会把 namespace 下的所有 Secret 全部拉取下来，这样其实暴露了更多的信息。推荐使用 GET 的方法，这样只获取你自己需要的那个 Secret。

## ServiceAccount
### ServiceAccount 介绍
ServiceAccount 首先是用于解决 pod 在集群里面的身份认证问题，身份认证信息是存在于 Secret 里面。

## Resource
### 容器资源配合管理
目前内部支持类型有三种：CPU、内存以及临时存储。如果用户觉得这三种不够，可以自己定义所需的资源，如 GPU。配置资源时，资源指定数量必须为整数。目前资源配置主要分成 request 和 limit 两种类型，一个是需要的数量，一个是资源的界限。CPU、内存以及临时存储都是在 container 下的 Resource 字段里进行一个声明。

### Pod 服务质量 (QoS) 配置
根据 CPU 对容器内存资源的需求，我们对 pod 的服务质量进行一个分类，分别是 Guaranteed、Burstable 和 BestEffort。

* Guaranteed：pod 里面每个容器都必须有内存和 CPU 的 request 以及 limit 的一个声明，且 request 和 limit 必须是一样的，这就是 Guaranteed；
* Burstable：Burstable 至少有一个容器存在内存和 CPU 的一个 request；
* BestEffort：只要不是 Guaranteed 和 Burstable，那就是 BestEffort。

服务质量是什么样的呢？资源配置好后，当这个节点上 pod 容器运行，比如说节点上 memory 配额资源不足，kubelet会把一些低优先级的，或者说服务质量要求不高的（如：BestEffort、Burstable）pod 驱逐掉。它们是按照先去除 BestEffort，再去除 Burstable 的一个顺序来驱逐 pod 的。

## SecurityContext

### SecurityContext 介绍

SecurityContext 主要是用于限制容器的一个行为，它能保证系统和其他容器的安全。这一块的能力不是 Kubernetes 或者容器 runtime 本身的能力，而是 Kubernetes 和 runtime 通过用户的配置，最后下传到内核里，再通过内核的机制让 SecurityContext 来生效。

SecurityContext 主要分为三个级别：

* 第一个是容器级别，仅对容器生效；
* 第二个是 pod 级别，对 pod 里所有容器生效；
* 第三个是集群级别，就是 PSP，对集群内所有 pod 生效。

权限和访问控制设置项，现在一共列有七项（这个数量后续可能会变化）：

1. 第一个就是通过用户 ID 和组 ID 来控制文件访问权限；
2. 第二个是 SELinux，它是通过策略配置来控制用户或者进程对文件的访问控制；
3. 第三个是特权容器；
4. 第四个是 Capabilities，它也是给特定进程来配置一个 privileged 能力；
5. 第五个是 AppArmor，它也是通过一些配置文件来控制可执行文件的一个访问控制权限，比如说一些端口的读写；
6. 第六个是一个对系统调用的控制；
7. 第七个是对子进程能否获取比父亲更多的权限的一个限制。

## InitContainer

### InitContainer 介绍

首先介绍 InitContainer 和普通 container 的区别，有以下三点内容：

1. InitContainer 首先会比普通 container 先启动，并且直到所有的 InitContainer 执行成功后，普通 container 才会被启动
2. InitContainer 之间是按定义的次序去启动执行的，执行成功一个之后再执行第二个，而普通的 container 是并发启动的；
3. InitContainer 执行成功后就结束退出，而普通容器可能会一直在执行。它可能是一个 longtime 的，或者说失败了会重启，这个也是 InitContainer 和普通 container 不同的地方

InitContainer 其实主要为普通 container 服务，比如说它可以为普通 container 启动之前做一个初始化，或者为它准备一些配置文件， 配置文件可能是一些变化的东西。再比如做一些前置条件的校验，如网络是否联通。

## 结束语

* ConfigMap 和 Secret: 首先介绍了 ConfigMap 和 Secret 的创建方法和使用场景，然后对 ConfigMap 和 Secret 的常见使用注意点进行了分类和整理。最后介绍了私有仓库镜像的使用和配置；
* Pod 身份认证: 首先介绍了 ServiceAccount 和 Secret 的关联关系，然后从源码角度对 Pod 身份认证流程和实现细节进行剖析，同时引出了 Pod 的权限管理(即 RBAC 的配置管理)；
* 容器资源和安全： 首先介绍了容器常见资源类型 (CPU/Memory) 的配置，然后对 Pod 服务质量分类进行详细的介绍。同时对 SecurityContext 有效层级和权限配置项进行简要说明；
* InitContainer: 首先介绍了 InitContainer 和普通 container 的区别以及 InitContainer 的用途。然后基于实际用例对 InitContainer 的用途进行了说明。


