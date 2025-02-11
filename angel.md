# 容器云知识复习

## 1，Pod

### 1.1 Pod是什么

pod是kubernetes最小的调度单元

> 1，可以处理超亲密关系调度，
> 
> 2， pod中的容器共享同一个NetworkNamespace、由infra容器hold住network Namespace，其他容器只需要加入这个容器，同一个pod中的容器可以通过
> 
> 3，可以声明同一个volume，容器间可以通过这个volume共享数据
> 
> 4，可以把pod比作虚拟机，容器是虚拟机中运行的用户进程

凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。

> 1，这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”
> 
> 2，凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod 级别的
> 
> 3，凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义
> 
> ```
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx
> spec:
>   hostNetwork: true
>   hostIPC: true
>   hostPID: true
>   containers:
>   - name: nginx
>     image: nginx
>   - name: shell
>     image: busybox
>     stdin: true
>     tty: true
> ```

> 在这个 Pod 中，我定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个 **Pod 里的所有容器**，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。

### 1.2     pod进阶使用

> **projectd volume** : 被kubernetes投射到容器中，类型有
> 
> Secret；ConfigMap；**Downward API；** ServiceAccountToken。
> 
> 注意： Downward API是pod启动钱就可以确定的东西

> **restart policy** :
> 
> **Always**在任何情况下，只要容器不在运行状态，就自动重启容器；（对于job类型执行结束的没有意义）
> 
> **OnFailure:** 只在容器 异常时才自动重启容器；
> 
> **Never:** 从来不重启容器。
> 
> > 1，只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod 就会保持 Running 状态，并进行容器重启。否则，Pod 就会进入 Failed 状态 。
> > 
> > 2，对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数.
> > 
> > 3，所以，假如一个 Pod 里只有一个容器，然后这个容器异常退出了。那么，只有当 restartPolicy=Never 时，这个 Pod 才会进入 Failed 状态。
> > 
> > 4，而其他情况下，由于 Kubernetes 都可以重启这个容器，所以 Pod 的状态保持 Running 不变。而如果这个 Pod 有多个容器，仅有一个容器异常退出，它就始终保持 Running 状态，哪怕即使 restartPolicy=Never。只有当所有容器也异常退出之后，这个 Pod 才会进入 Failed 状态。

## 1.3 监控监测

> **livenessProbe**：
> 
> **readinessProbe:** readinessProbe 检查结果的成功与否，决定的这个 Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期

## 1.4 PodPreSet

> 1，PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。
> 
> 2，如果你定义了同时作用于一个 Pod 对象的多个 PodPreset
> 
> 3，Kubernetes 项目会帮你合并（Merge）这两个 PodPreset 要做的修改。而如果它们要做的修改有冲突的话，这些冲突字段就不会被修改。

### 1.5 Pod 生命周期

> 1， Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
> 
> 2，Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
> 
> 3，Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
> 
> 4，Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
> 
> 5，Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。
> 
> > 更进一步地，Pod 对象的 Status 字段，还可以再细分出一组 Conditions。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及 Unschedulable。它们主要用于描述造成当前 Status 的具体原因是什么。

### 1.6 Lifecycle 字段

> 1，postStart ---它指的是，在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart 定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。
> 
> 2，如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。
> 
> 3，preStop 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

### 1.5 QOS

#### QoS 级别

> 1，**Guaranteed 类别:** 当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 Guaranteed 类别,当这个 Pod 创建之后，它的 qosClass 字段就会被 Kubernetes 自动设置为 Guaranteed
> 
> 2，**Burstable 类别** ：当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别
> 
> 3，**BestEffort：** 如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort

#### Eviction

QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的。

当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction

> Eviction 在 Kubernetes 里其实分为 Soft 和 Hard 两种模式
> 
> > Soft Eviction 允许你为 Eviction 过程设置一段“优雅时间”
> > 
> > Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始
> > 
> > > 1，Kubernetes 计算 Eviction 阈值的数据来源，主要依赖于从 Cgroups 读取的值，以及使用 cAdvisor 监控到的数据。
> > > 
> > > 2，当宿主机的 Eviction 阈值达到后，就会进入 MemoryPressure 或者 DiskPressure 状态，从而避免新的 Pod 被调度到这台宿主机上
> 
> Eviction 发生的时候，kubelet 具体会挑选哪些 Pod 进行删除操作，就需要参考这些 Pod 的 QoS 类别
> 
> > 1，首当其冲的，自然是 BestEffort 类别的 Pod。
> > 
> > 2，其次，是属于 Burstable 类别、并且发生“饥饿”的资源使用量已经超出了 requests 的 Pod。
> > 
> > 3，最后，才是 Guaranteed 类别。并且，Kubernetes 会保证只有当 Guaranteed 类别的 Pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作。
> > 
> > > 对于同 QoS 类别的 Pod 来说，Kubernetes 还会根据 Pod 的优先级来进行进一步地排序和选择
> 
> **强烈建议将 DaemonSet 的 Pod 都设置为 Guaranteed 的 QoS 类型**

#### CPUset

你可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力

> 由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。事实上，cpuset 方式，是生产环境里部署在线应用类型的 Pod 时，非
> 
> 常常用的一种方式

> 实现：
> 
> > 1，首先，你的 Pod 必须是 Guaranteed 的 QoS 类型；
> > 
> > 2，然后，你只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值即可。

## 2，容器运行时

![](https://static001.geekbang.org/resource/image/91/03/914e097aed10b9ff39b509759f8b1d03.png)

### 2.1 kubelet SyncLoop

SyncLoop 又是如何根据 Pod 对象的变化，来进行容器操作的呢

> #### kubelet 的工作核心，就是一个控制循环，即：SyncLoop（图中的大圆圈）
> 
> > 1，Pod 更新事件；
> > 
> > 2，Pod 生命周期变化；
> > 
> > 3，kubelet 本身设置的执行周期；
> > 
> > 4，定时的清理事件。
> 
> #### kubelet 还负责维护着很多很多其他的子控制循环（也就是图中的小圆圈）。
> 
> > 1，这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager 等等。
> > 
> > 2，不难想到，这些控制循环的责任，就是通过控制器模式，完成 kubelet 的某项具体职责。
> > 
> > 3，比如 Node Status Manager，就负责响应 Node 的状态变化，然后将 Node 的状态收集起来，并通过 Heartbeat 的方式上报给 APIServer。
> > 
> > 4，再比如 CPU Manager，就负责维护该 Node 的 CPU 核的信息，以便在 Pod 通过 cpuset 的方式请求 CPU 核的时候，能够正确地管理 CPU 核的使用量和可用量。
> 
> #### kubelet 也是通过 Watch 机制，监听了与自己相关的 Pod 对象的变化
> 
> > kubelet 会启动一个名叫 Pod Update Worker 的、单独的 Goroutine 来完成对 Pod 的处理工作。
> > 
> > > 1， 如果是 ADD 事件的话，kubelet 就会为这个新的 Pod 生成对应的 Pod Status，检查 Pod 所声明使用的 Volume 是不是已经准备好。然后，调用下层的容器运行时（比如 Docker），开始创建这个 Pod 所定义的容器。
> > > 
> > > 2，如果是 UPDATE 事件的话，kubelet 就会根据 Pod 对象具体的变更情况，调用下层容器运行时进行容器的重建工作。

### 2.2 kubelet 如何创建容器

kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker 的 API，而是通过一组叫作 CRI（Container Runtime Interface，容器运行时接口）的 gRPC 接口来间接执行的。

> ![](https://static001.geekbang.org/resource/image/51/fe/5161bd6201942f7a1ed6d70d7d55acfe.png)
> 
> 1，当 Kubernetes 通过编排能力创建了一个 Pod 之后，调度器会为这个 Pod 选择一个具体的节点来运行。这时候，kubelet 当然就会通过前面讲解过的 SyncLoop 来判断需要执行的具体操作，比如创建一个 Pod。那么此时，kubelet 实际上就会调用一个叫作 GenericRuntime 的通用组件来发起创建 Pod 的 CRI 请求
> 
> 2，如果你使用的容器项目是 Docker 的话，那么负责响应这个请求的就是一个叫作 dockershim 的组件。它会把 CRI 请求里的内容拿出来，然后组装成 Docker API 请求发给 Docker Daemon。
> 
> 而更普遍的场景，就是你需要在每台宿主机上单独安装一个负责响应 CRI 的组件，这个组件，一般被称作 CRI shim。顾名思义，CRI shim 的工作，就是扮演 kubelet 与容器项目之间的“垫片”（shim）。所以它的作用非常单一，那就是实现 CRI 规定的每个接口，然后把具体的 CRI 请求“翻译”成对后端容器项目的请求或者操作。

### 2.3     CRI

> ![](https://static001.geekbang.org/resource/image/70/38/7016633777ec41da74905bfb91ae7b38.png)
> 
> CRI 机制能够发挥作用的核心，就在于每一种容器项目现在都可以自己实现一个 CRI shim，自行对 CRI 请求进行处理。这样，Kubernetes 就有了一个统一的容器抽象层，使得下层容器运行时可以自由地对接进入 Kubernetes 当中
> 
> 这里的 CRI shim，就是容器项目的维护者们自由发挥的“场地”了。而除了 dockershim 之外，其他容器运行时的 CRI shim，都是需要额外部署在宿主机上的。
> 
> CNCF 里的 containerd 项目，就可以提供一个典型的 CRI shim 的能力，
> 
> > 即：将 Kubernetes 发出的 CRI 请求，转换成对 containerd 的调用，然后创建出 runC 容器。
> > 
> > 而 runC 项目，才是负责执行我们前面讲解过的设置容器 Namespace、Cgroups 和 chroot 等基础操作的组件。所以，这几层的组合关系，可以用如下所示的示意图来描述。
> > 
> > ![](https://static001.geekbang.org/resource/image/62/3d/62c591c4d832d44fed6f76f60be88e3d.png)
> 
> 而作为一个 CRI shim，containerd 对 CRI 的具体实现，又是怎样的呢？
> 
> ![](https://static001.geekbang.org/resource/image/f7/16/f7e86505c09239b80ad05aecfb032e16.png)
> 
> > #### CRI 分为两组:
> > 
> > > 第一组，是 RuntimeService。它提供的接口，主要是跟容器相关的操作。比如，创建和启动容器、删除容器、执行 exec 命令等等。
> > > 
> > > 第二组，则是 ImageService。它提供的接口，主要是容器镜像相关的操作，比如拉取镜像、删除镜像等等。
> > 
> > #### CRI 设计的一个重要原则，就是确保这个接口本身，只关注容器，不关注 Pod
> > 
> > > 第一，Pod 是 Kubernetes 的编排概念，而不是容器运行时的概念。所以，我们就不能假设所有下层容器项目，都能够暴露出可以直接映射为 Pod 的 API。
> > > 
> > > 第二，如果 CRI 里引入了关于 Pod 的概念，那么接下来只要 Pod API 对象的字段发生变化，那么 CRI 就很有可能需要变更。而在 Kubernetes 开发的前期，Pod 对象的变化还是比较频繁的，但对于 CRI 这样的标准接口来说，这个变更频率就有点麻烦了。
> > 
> > #### **CRI 里还是有一组叫作 RunPodSandbox 的接口的**
> > 
> > > 1，这个 PodSandbox，对应的并不是 Kubernetes 里的 Pod API 对象，而只是抽取了 Pod 里的一部分与容器运行时相关的字段。
> > > 
> > > 2，比如 HostName、DnsConfig、CgroupParent 等
> > > 
> > > 3，**PodSandbox 这个接口描述的，其实是 Kubernetes 将 Pod 这个概念映射到容器运行时层面所需要的字段，或者说是一个 Pod 对象子集。**
> > 
> > #### cri shim工作
> > 
> > > 2，在具体的 CRI shim 中，这些接口的实现是可以完全不同的。比如，如果是 Docker 项目，dockershim 就会创建出一个名叫 foo 的 Infra 容器（pause 容器），用来“hold”住整个 Pod 的 Network Namespace；如果是基于虚拟化技术的容器，比如 Kata Containers 项目，它的 CRI 实现就会直接创建出一个轻量级虚拟机来充当 Pod
> > > 
> > > 3，在 RunPodSandbox 这个接口的实现中，你还需要调用 networkPlugin.SetUpPod(…) 来为这个 Sandbox 设置网络。这个 SetUpPod(…) 方法，实际上就在执行 CNI 插件里的 add(…) 方法，也就是我在前面为你讲解过的 CNI 插件为 Pod 创建网络，并且把 Infra 容器加入到网络中的操作
> > > 
> > > 4，kubelet 继续调用 CreateContainer 和 StartContainer 接口来创建和启动容器 A、B。对应到 dockershim 里，就是直接启动 A，B 两个 Docker 容器。所以最后，宿主机上会出现三个 Docker 容器组成这一个 Pod，如果是 Kata Containers 的话，CreateContainer 和 StartContainer 接口的实现，就只会在前面创建的轻量级虚拟机里创建两个 A、B 容器对应的 Mount Namespace，最后在宿主机上，只会有一个叫作 foo 的轻量级虚拟机在运行
> > 
> > #### **Streaming API**
> > 
> > > CRI shim 还有一个重要的工作，就是如何实现 exec、logs 等接口。这些接口跟前面的操作有一个很大的不同，就是这些 gRPC 接口调用期间，kubelet 需要跟容器项目维护一个长连接来传输数据。这种 API，我们就称之为**Streaming API**
> > 
> > > 1，CRI shim 里对 Streaming API 的实现，依赖于一套独立的 Streaming Server 机制
> > > 
> > > 2，可以看到，当我们对一个容器执行 kubectl exec 命令的时候，这个请求首先交给 API Server，然后 API Server 就会调用 kubelet 的 Exec API。
> > > 
> > > 3，kubelet 就会调用 CRI 的 Exec 接口，而负责响应这个接口的，自然就是具体的 CRI shim
> > > 
> > > 4，CRI shim 并不会直接去调用后端的容器项目（比如 Docker ）来进行处理，而只会返回一个 URL 给 kubelet。这个 URL，就是该 CRI shim 对应的 Streaming Server 的地址和端口
> > > 
> > > 5，kubelet 在拿到这个 URL 之后，就会把它以 Redirect 的方式返回给 API Server。所以这时候，API Server 就会通过重定向来向 Streaming Server 发起真正的 /exec 请求，与它建立长连接
> > 
> > > 这个 Streaming Server 本身，是需要通过使用 SIG-Node 为你维护的 Streaming API 库来实现的。并且，Streaming Server 会在 CRI shim 启动时就一起启动。此外，Stream Server 这一部分具体怎么实现，完全可以由 CRI shim 的维护者自行决定。比如，对于 Docker 项目来说，dockershim 就是直接调用 Docker 的 Exec API 来作为实现的
> > 
> > ![](https://static001.geekbang.org/resource/image/a8/ef/a8e7ff6a6b0c9591a0a4f2b8e9e9bdef.png)

### 2.4 CSI

> ### External Components
> 
> > external provisioner
> > 
> > Driver register
> > 
> > external attacher
> 
> ### CSI 插件服务
> 
> > CSI  Identity
> > 
> > CSI Controller
> > 
> > CSI Node

### 2.5 CNI

### 2.6 Deployment 对象创建过程

> kubectl使用yaml deployment 描述文件创建deployment对象
> 
> #### 客户端kubectl
> 
> > 1，**客户端验证**——kubectl create -f xx.yaml 触发kubectl执行一些客户端验证操作，以确保不合法的请求（如不支持的资源类型或者格式错误的镜像名称）会快速失败，不会发送给kube-apiserver，通过减少不必要的负载提高系统性能；
> > 
> > 2，**客户端封装请求**——kubectl对向apiserver发起的请求进行封装，kubectl使用生成器（Generator）来构造https请求（处理序列化的抽象概念）；
> > 
> > 3，**版本协商**—— 根据yaml文件中apiVersion，如deployment 的group是apps version是v1beta2 ，找到合适的url路径；
> > 
> > 4，**发送请求**——通过上述的url路径和请求body发送请求，一旦获得相应，kubectl就打印成功日志
> 
> #### apiserver
> 
> > 5，**认证客户端-authentication**-通过tls或者其他方式验证请求是否是合法的客户端
> > 
> > 6，**鉴权客户端-authorization**-验证客户端是否有权限，执行此请求的操作
> > 
> > 7，**admission controller**-此步是写etcd最后一步，主要确保请求满足集群更加广泛的需求，默认的比如resourceQuota，还可以自定义admission webhook；
> > 
> > 8，**反序列化**——：对客户端请求进行反序列化，得到构建运行时对象，保存到etcd中；
> > 
> > 9，**initailize**——：和资源类型相关联的控制器，会在资源对外可见之前，执行某些逻辑，如为对象注入边车容器；
> > 
> > 10，执行完initialize 操作或没有设置initialize后，此对象将对外课件。Deployment对象被存入到etcd中；
> 
> #### Controller
> 
> > 11，**Deployment Controller** ——Deployment 被保存到etcd中，Deployment Controller会检测到deployment对象创建，并将该资源对象添加到工作队列中，然后开始处理这个资源对象（通过使用便签选择器查询kube-apiserver来检查deployment是否有关联的RS或者Pod记录，意识到没有关联的RS或者Pod，就会创建RS资源对象，为其分配一个便签选择器版本为1，RS的PodSpec字段从Deployment的manifest以及其他相关元数据复制而来）
> > 
> > 12，**ReplicaSet** —— Deployment Controller创建了第一个ReplicaSet，但创建ReplicaSet时，RS Controller检查新的ReplicaSet的状态，并检查当前状态与期望状态之间存在差别，通过调整Pod的副本数来达到期望的状态。（Pod是批量创建，每次成功以slow start操作加倍）
> > 
> > **Owner Reference** —— TODO
> 
> ### Scheduler
> 
> > 当所以Controller正常运行后，etcd中保存一个Deployment，一个ReplicaSet和一个Pod。（**pod的资源现在还处于Pending状态，表示还没有调度**）
> > 
> > scheduler的作用是将待调度的Pod按照特定的算法和调度策略绑定到集群的某个合适的Node，并将其写入etcd中
> > 
> > 13，scheduler监听nodeName字段为空的新创建的Pod，当Scheduler监听到Pod创建之后，将Pod添加到一个优先级队列中。
> > 
> > 14，scheduler 另一个goroutine会不断地从优先级队列中取出Pod对象，可以并发的调用predicate算法，predicate算法主要是过滤掉不适合pod运行的节点；
> > 
> > 15，14中得到的node节点作为priority算法的输入，priority算法会对节点进行打分，得分最高的节点将作为Pod要运行的节点。
> > 
> > 16，**bind**—— 一旦找到合适的节点，scheduler就会创建一个Binding对象该对象的Name和Uid与Pod相匹配，并且其ObjectReference字段包含所选择节点的名称，如何以Post请求发送给apiserver;
> > 
> > 一旦scheduler将pod调度到某个节点上，该节点的kubelet就会接管该Pod
> 
> #### kubelet
> 
> > kubelet 每隔20秒（可以自定义）向kube-apiserver通过NodeName获取自身Node上所有要运行Pod的清单，一旦获取到这个清单，通过自己内部的缓存来比较新增加的Pod，如果有差异，就开始同步Pod列表；
> > 
> > 17，如果pod正在创建，kubelet会记录一些在Prometheus中用于跟踪Pod启动延时的指标；
> > 
> > 18，生产一个PodStatus对象，表示pod当前状态，如Running，Pending，Successed,Failed和Unknown等
> > 
> > > a.串行执行一系列pod同步处理器（PodSyncHandlers），**所用处理器**认为pod不该再节点上运行，pod的phase会变成PodFailed，将Pod从节点上驱逐出去。Pod失败重试的时间超过activeDeadlineSecends，pod也会被驱逐
> > > 
> > > b.Pod的Phase值由init容器和应用容器的状态共同决定，至少有一个容器处于等待阶段，则phase值为Pending
> > > 
> > > c.Pod的Condition状态由Pod中所有容器决定，容器还没有被容器运行时创建的时候，PodReady的状态被设置为False
> > 
> > 19，生成PodStatus之后，kubelet会将它发送到Pod的状态管理器，该管理器任务通过Apiserver异步更新到etcd中
> > 
> > 20，运行一系列准入处理器来确保该Pod是否具有相应的权限，被准入控制器拒绝的Pod将一直保持Pending状态；如果kubelet启动时制定了cgroup-per-qos参数，kubelet就会为该Pod创建cgroup并进行相应的资源限制；
> > 
> > 21，为Pod创建相应的目录，包括Pod的目录(/var/run/kubelet/pod/)，该pod的卷目录（/volumes）和该Pod的插件目录（/plugins）
> > 
> > 22，卷挂载管理器会关注Spec.Volumes中定义的相关数据卷，等待是否挂载成功
> > 
> > 23，从Apiserver中检索Spec.ImagePullSecrets中定义的所有Secret，然后注入到容器中
> > 
> > 24，通过容器运行时接口（CRI）开始启动容器。
> 
> #### CRI和pause容器
> 
> > 25，第一层启动Pod，kubelet会通过（RPC）协议调用RunPodSandbox。如果容器运行是docker，创建sandbox时首先创建的是pause容器，**pause**容器作为同一个Pod中所有容器的基础，为Pod中的其他容器提供Pod级别资源，这些资源都是Linux空间（比如network Namespce，IPC Namespce，PIDNamespace）
> > 
> > **Pod**——Pod提供一种方式来管理所有这些命名空间并运行业务容器来共享它们。
> > 
> > **网络命令空间好处**——localhost可以直接访问，pid命名空间中进程形成一个树状结构，子进程变成孤儿进程，会被init进程进行收养并回收资源。
> > 
> > **sandbox**——描述一组容器，在kubernetes中表示一个Pod，如果是基于hypervisor，sandbox可能指虚拟机
> > 
> > pause容器准备好，开始检查磁盘状态并开始启动业务容器;
> > 
> > **Pod骨架**——一个共享所有命名空间以允许业务容器在同一个Pod里进行通信的pause容器
> > 
> > 26，**容器网络**—— 创建网络的工作是交给CNI插件，CNI是一个抽象的，允许不同网络提供商为容器提供不同的网络实现。通过json配置文件（默认在/etc/cni/net.d路径下）中的数据传输给相关的CNI二进制文件（默认在/opt/cni/bin）下，cni插件可以给pause容器配置相关网络，然后Pod中其他容器都是使用pause容器的网络。
> > 
> > CNI还可以通过CNI_ARGS环境变量为Pod制定其他的元数据，包括Pod名称和命令空间
> > 
> > ```
> > "cniVersion": "0.3.1",
> >     "name": "bridge",
> >     "type": "bridge",
> >     "bridge": "cnio0",
> >     "isGateway": true,
> >     "ipMasq": true,
> >     "ipam": {
> >         "type": "host-local",
> >         "ranges": [
> >           [{"subnet": "${POD_CIDR}"}]
> >         ],
> >         "routes": [{"dst": "0.0.0.0/0"}]
> >     }
> > }
> > ```
> > 
> > #### 一个CNI创建的过程 以bridge为例
> > 
> > > a.该插件首先在根网络命名空间（也就是宿主机的网络命名空间）设置本地linux网桥，以便为所有容器提供网络服务；
> > > 
> > > b.将一个网络接口（veth设备对的一端）插入到pause容器的网络命名空间中，并将另一端连接到网桥;(veth像一根管道，一端连着容器的网络命名空间，另一端连接着根网络命名空间，数据包就在管道中进行传播)
> > > 
> > > c.使用json中指定的IPAM Plugin会为pause容器的网络接口分配一个IP并设置相应的路由，现在Pod就有了IP;
> > > 
> > > IPAM plugin的工作方式和CNI plugins类似：通过二进制文件调用并具有标准化的接口，**每一个IPAM plugin都必须确定容器网络接口的IP、子网、网关以及路由，并将信息返回给CNI插件**
> > > 
> > > d.kubelet会将集群内部的DNS服务器的Cluster IP地址传给CNI插件，然后CNI插件将他们写入到/etc/resolv.conf文件中
> > > 
> > > e.完成上述步骤，cni插件就会将操作的结果以json的方式返回给kubelet
> > 
> > #### 跨主机网络
> > 
> > > 通常使用overlay网络来跨主机通信，动态的同步多个主机间路由的方法
> > 
> > #### 容器启动
> > 
> > 所有网络配置完成之后，就是启动业务容器
> > 
> > 一旦sandbox完成初始化并处于active状态，kubelet就开始创建容器，首先启动PodSpec中定义的init容器，然后再启动业务容器
> > 
> > > a.拉去容器的镜像，如果是私有仓库的镜像，就会利用PodSpec中指定的Secret来拉取该镜像；
> > > 
> > > b.通过CRI接口创建容器。kubelet向PodSpec中填充一个ContainerConfig数据结构（定义了命令、镜像、标签、挂载卷、设备、环境变量等），通过protobufs发送给CRI接口。
> > > 
> > > > 对于docker来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给Dockerd守护进程。在这个过程他会将一些元数据标签（例如容器类型，日志路径，sandboxID等）添加到容器中
> > > 
> > > c.接下来会使用CPU管理来约束容器，他是用UpdateContainerResources CRI方法将容器分配给本节点上的CPU资源池
> > > 
> > > d.容器开始启动
> > > 
> > > e.如果有钩子，在容器启动之后就会运行这些钩子。（**如果PostStart Hook启动时间过长、挂起或者失败，容器将永远不会变成running状态**）

## 3，apiserver是什么

apiserver 分为 kube-apiserver 、aggregator-apiserver、

[apiserver]([kube-apiserver · Kubernetes指南](https://feisky.gitbooks.io/kubernetes/content/components/apiserver.html))

> kube-apiserver 是kubernetes中最重要的组件主要：
> 
> > 1，提供管理集群的restful api，包括鉴权授权、数据校验、集群状态的变更等
> > 
> > 2，提供与其他模块之间的数据交换和通信枢纽（集群中只有apiserver可以访问etcd）
> 
> kube-apiserver GVK
> 
> > ![img](https://feisky.gitbooks.io/kubernetes/content/components/assets/API-server-space.png)
> 
> 访问控制
> 
> 每个kubenetes api请求会经历多个阶段才会被接受，包括认证、授权、准入控制等
> 
> ![](https://feisky.gitbooks.io/kubernetes/content/components/images/access_control.png)
> 
> > 1，认证：开启tls时，所有请求都需要进行认证
> > 
> > 2，授权: 判断用户是否有权限对这个api对象操作，RBAC
> > 
> > 3，准入控制：仅对创建、更新、删除或连接（如代理）等有效，而**对读操作无效**
> 
> ![](https://feisky.gitbooks.io/kubernetes/content/components/images/kube-apiserver.png)
> 
> 以 `/apis/batch/v2alpha1/jobs` 为例，GET 请求的处理过程如下图所示：
> 
> ![img](https://feisky.gitbooks.io/kubernetes/content/components/assets/API-server-flow.png)
> 
> POST 请求的处理过程为：
> 
> ![img](https://feisky.gitbooks.io/kubernetes/content/components/assets/API-server-storage-flow.png)

## 4，scheduler

### #### TODO [41 | 十字路口上的Kubernetes默认调度器-极客时间](https://time.geekbang.org/column/article/69890)

### 4.1 scheduler 是什么

默认调度器的主要职责，就是为一个新创建出来的 Pod，寻找一个最合适的节点（Node）

> 1，从集群所有节点中，找到所有能适合运行pod的节点，使用predicate算法检查每个Node；
> 
> 2，从第一步中结果中，选择找出最适合运行pod的节点，使用priority算法为每个节点打分。
> 
> 最终结果结果就是等分最高的那个

### 4.2 调度器的工作机制

> 1，控制循环Informer Path：scheduler通过informer机制监听pod的变化，当nodeName为pod创建之后，scheduler通过pod informer 的handler，将pod添加进调度队列中；
> 
> 2，控制循环schedulering Path：是调度器负责pod调度的主循环 ,主要逻辑是不断从队列中弹出一个pod
> 
> > 1 ，调用predicate算法对全部node进行过滤，得到了一组过滤之后的node，predicate算法需要的node信息都是**从scheduler cache中**直接拿到的
> > 
> > 2，调用priority算法对上述得到的node进行打分，0-10，得分最高的是本次调度的结果
> 
> 3，调度完成之后，调度器将设置pod的nodeName字段，完成成为Bind的阶段。

> **Assume** ：基于乐观假设的apiserver对象更新方式，为了不再关键路径访问远程访问APIServer，kubernetes在bind阶段，只会更新scheduler cache中的
> 
> assume之后，调度器会创建一个Goroutine异步的向apiserver发起apiserver的更新操作，来完成bind操作。即使失败了，也没有太大关系，等scheduler cache同步之后就恢复正常了。
> 
> **kubelet二次确定**：当一个pod完成bind阶段之后，kubelet还会进行一次admit操作，就是执行一次genericPredicate操作，包括端口是否冲突，资源是否够用等

> scheduler 重要设计
> 
> 1，cache化；2，乐观绑定；3，无锁化
> 
> 无锁化：
> 
> > 1，schedule Path中，调度器会启动多个Goroutine去并发执行predicate操作，priority也会使用mapreduce的方式执行。
> > 
> > 2，schedule过程，只有再**调度队列**和**schedule cache**中使用到锁，再schedule Path中是**没有使用锁**的。

> Scheduler Framework
> 
> ![](https://static001.geekbang.org/resource/image/fd/cd/fd17097799fe17fcbc625bf178496acd.jpg)
> 
> > 在调度器的生命周期的几个关键节点上，为用户暴露出可以进行扩展和实现的接口，从而实现用户 自定义调度
> > 
> > 问题
> > 
> > > 这样的 Scheduler Framework 也有一个不小的问题，那就是一旦这些插入点的接口设计不合理，就会导致整个生态没办法很好地把这个插件机制使用起来。而与此同时，这些接口本身的变更又是一个费时费力的过程，一旦把控不好，就很可能会把社区推向另一个极端，即：Scheduler Framework 没法实际落地，大家只好都再次 fork kube-scheduler。总结

### 4.3 自己的话总结scheduler

> scheduler的作用是为一个新创建的pod，寻找一个合适的节点，需要经过两个控制循环。**（添加predicate priority具体过程）**
> 
> > 1，第一个控制循环，通过informer机制监听pod的创建，当一个nodeName为空的pod创建时，就会触发informer的handler，将pod放入到一个优先级队列中。
> > 
> > 2，第二个控制循环，就从队列中弹出一个pod，会启动多个Goroutine，调用predicate算法对node节点进行筛选，筛选出来的node节点用作priority阶段的数据，priority算法将对筛选出来的node进行打分，选出得分最高的node进行返回。
> > 
> > 3，bind阶段，将pod的nodeName字段设置为选定的节点，此时只是更新scheduler cache中的信息，调度器会启动一个goroutine异步的向apiserver更新pod信息，即使失败，在下次schedule cache同步就会解决。
> > 
> > 4，kubelet watch到一个pod将运行在本节点，会执行二次确认判断pod的端口是否适合、资源是否够用等。

## 5，controller是什么

## 6，网络相关问题

## 6.1 flannel

#### 6.1.1 udp

#### 6.1.2 vxlan

#### 6.1.3 hostgw

## 6.2 calico

##### 6.2.1 ipip

#### 6.2.2 vxlan

#### 6.2.3 bgp

## 7，etcd相关

### etcd基础架构介绍

![](https://static001.geekbang.org/resource/image/45/bb/457db2c506135d5d29a93ef0bd97e4bb.png?wh=1920*1229)

> **Client层** ——主要提供client v2和v3两个大版本API客户端库，提供了简洁易用的API，同时支持**负载均衡**、**节点故障转移**、**可极大降低业务使用etcd**、提高开发率、服务可用性；
> 
> **API层** —— API网络层包括**client访问server**和**server之间的通信协议**
> 
> > **a**.client访问etcd server的API分为v2和v3两个版本；
> > 
> > > v2 API使用HTTP/1.x协议；
> > > 
> > > v3 API使用gRPC协议，同时v3 API通过etcd grpc-gateway组件也支持HTTP/1.x协议
> > 
> > **b**.etcd server之间的通信协议，是指节点间Raft算法实现数据复制和Leader选举等功能时使用的http协议
> 
> **Raft算法层** ——Raft算法实现了**Leader选举**，**日志复制**，**ReadIndex**等核心算法特征，用于**保障etcd多个节点间的数据一致性**、**提升服务可用性**，是etcd的基石和亮点
> 
> **逻辑层** ——etcd核心特性实现层，如典型的**KVServer模块**、**MVCC模块**、**Auth模块**、**Lease租约模块**、**Compator压缩模块**；其中**MVCC模块**主要由**treeIndex模块**和**boltdb**模块组成
> 
> **存储层** ——存储层包括**预写日志WAL模块**、**快照Snapshot模块**、**boltdb模块**；
> 
> > **WAL**——保障etcd Crash之后数据不会丢失；
> > 
> > **boltdb**——保存**集群元数据**和**用户写入的数据**

### 7.1 etcd的读请求.

> ![](https://static001.geekbang.org/resource/image/45/bb/457db2c506135d5d29a93ef0bd97e4bb.png?wh=1920*1229)
> 
> #### client
> 
> ```
> etcdctl get hello --endpoints http://127.0.0.1:2379  
> ```
> 
> 执行上诉命令，是如何工作的？
> 
> > a.etcdctl 会对命令中的参数进行解析。我们来看下这些参数的含义，其中，参数“get”是请求的方法，它是 KVServer 模块的 API；
> > 
> > > 1 . **endpoints”是我们后端的 etcd 地址，通常，生产环境下中需要配置多个 endpoints，这样在 etcd 节点出现故障后，client 就可以自动重连到其它正常的节点，从而保证请求的正常执行**
> > > 
> > > 2.etcdctl使用clientv3库来访问etcd server,**client v3库基于gRPC client API封装了操作etcd KVServer、Cluster、Auth、Lease、Watch等模块的API，同时还包括负载均衡、监控探测、故障切换的特性**
> > 
> > b.解析完请求中的参数.etcdctl会创建一个clientv3库对象，使用kvServer模块的API来访问etcd server
> > 
> > c.**负载均衡**——为get hello请求寻找一个合适的etcd server节点；
> > 
> > > etcd 3.4中，clientv3库采用round robin，依次轮询endpoint列表中的一个endpoint访问，建立长链接
> > > 
> > > 关于负载均衡算法，你需要特别注意以下两点。
> > > 
> > > > 如果你的 client 版本 <= 3.3，那么当你配置多个 endpoint 时，负载均衡算法仅会从中选择一个 IP 并创建一个连接（Pinned endpoint），这样可以节省服务器总连接数。但在这我要给你一个小提醒，在 heavy usage 场景，这可能会造成 server 负载不均衡。
> > > > 
> > > > 在 client 3.4 之前的版本中，负载均衡算法有一个严重的 Bug：如果第一个节点异常了，可能会导致你的 client 访问 etcd server 异常，特别是在 Kubernetes 场景中会导致 APIServer 不可用。不过，该 Bug 已在 Kubernetes 1.16 版本后被修复。
> > 
> > d.选择好etcd server节点，client就可以调用etcd server的KVserver模块的Range RPC方法，把请求发送给etcd server；
> > 
> > > client 和 server 之间的通信，使用的是基于 HTTP/2 的 gRPC 协议。相比 etcd v2 的 HTTP/1.x，HTTP/2 是基于二进制而不是文本、支持多路复用而不再有序且阻塞、支持数据压缩以减少包大小、支持 server push 等特性。因此，基于 HTTP/2 的 gRPC 协议具有低延迟、高性能的特点，有效解决了我们在上一讲中提到的 etcd v2 中 HTTP/1.x 性能问题
> 
> #### KVServer
> 
> client发送range rpc请求到server之后，就进入KVServer模块
> 
> > etcd 提供了丰富的 metrics、日志、请求行为检查等机制，可记录所有请求的执行耗时及错误码、来源 IP 等，也可控制请求是否允许通过，比如 etcd Learner 节点只允许指定接口和参数的访问，帮助大家定位问题、提高服务可观测性等，而这些特性是怎么非侵入式的实现呢？**就是拦截器**
> 
> #### 拦截器
> 
> 拦截器提供执行一个请求前后的hook能力，包括debug日志、metrics统计、对etcd learn节点请求接口和参数限制，还包括：
> 
> > - 要求执行一个操作前集群必须有leader
> > 
> > - 请求延时超过阈值的，打印包含来源IP的慢查询日志
> > 
> > server接受到client range rpc请求后，根据serverName和RPC Method将请求转发到对应的handler实现，handler首先会将上述拦截器串行执行
> 
> #### 串行读与线性读
> 
> ![](https://static001.geekbang.org/resource/image/cf/d5/cffba70a79609f29e1f2ae1f3bd07fd5.png?wh=1920*1074)
> 
> 问题：在多节点部署的情况下，如何保证**任意时间点**读出来的都是**一致**的？什么情况会**读到旧数据**
> 
> ##### 写流程 简述
> 
> > - 当client发起一个更新hello为world的请求
> > 
> > - 若是leader收到写请求，他会将请求持久化到WAL日志，并广播给各个节点
> > 
> > - 若一半以上节点持久化成功，则该请求的日志条目被标记为已提交
> > 
> > - etcd server模块异步的重Raft模块获取已提交的日志条目，应用到状态机（boltdb等）
> 
> ##### 读旧数据的情况
> 
> > 若client发起一个读取hello读请求，假设直接从状态机中读取，连接的节点是C节点，若C节点IO出现波动，导致应用应提交的日志条目很慢，则会出现更新hello为world的写命令，在client读hello的时候还没有提交到状态机，就可能读取到旧数据。
> > 
> > 多个节点的etcd集群中，各个节点状态机数据一致性存在差异
> > 
> > **不同业务场景对数据新旧程度容忍度不同**
> > 
> > > - 有的可以容忍数据落后几秒或者几分钟
> > >   
> > >   > 旁路数据统计服务，每分钟统计etcd中服务、配置信息，对数据实时性要求不高，
> > >   > 
> > >   > **串行读**——直接读取状态机数据，无法raft协议与集群交互的模式
> > >   > 
> > >   > ,特点具有低延时、高吞吐
> > > 
> > > - 有的就必须读到反映集群共识的最新数据
> > >   
> > >   > **线性读**——需要经过Raft协议模块，反应的集群共识，在延时性和吞吐量上略微差一点
> 
> ##### 线性读
> 
> ![](https://static001.geekbang.org/resource/image/1c/cc/1c065788051c6eaaee965575a04109cc.png?wh=1920*1095)
> 
> #### ReadIndex
> 
> > 线性读通过ReadIndex机制保证数据一致性
> > 
> > 其他一致性读请求：
> > 
> > > 通过走一遍raft协议保证一致性
> > 
> > 总体流程
> > 
> > > 1. KVServer模块收到线性读请求之后，向raft模块发起ReadIndex请求；
> > > 
> > > 2. readIndex处理
> > >    
> > >    > a.当收到一个读请求，它会从Leader获取集群最新的一提交的日志索引（Committed Index）
> > >    > 
> > >    > b.C节点会等待，直到状态机已应用的索引（applied index）大于等于Leader的已提交的索引（Committed Index），然后**通知读请求，数据已经赶上Leader，你可以状态机中访问数据了**
> > >    > 
> > >    > c.Raft模块将Leader最新已提交日志索引封装在流程四的ReadState结构体，通过Channel层层返回给线性读模块，线性读模块等待本节点状态机追赶上Leader进度；
> > > 
> > > 3. 追赶完成后，就通知KVServer模块，与状态机中的MVCC模块进行交互
> 
> #### MVCC
> 
> > MVCC是由**内存树形索引模块（treeIndex)** 和**嵌入式的KV持久化储存库boltdb**组成
> 
> > etcd如何保存一个key的多个历史版本
> > 
> > > - 每次修改操作，生成一个新的版本号（revision），以版本号为key，value是用户key-value等信息组成的结构体
> > > 
> > > - boltdb的key是全局递增的版本号（revision）,value是用户key、value等字段组合成的结构体，然后通过treeindex模块来保存用户key和版本号的映射关系
> > 
> > treeIndex和boltdb关系
> > 
> > > ![](https://static001.geekbang.org/resource/image/4e/a3/4e2779c265c1da1f7209b5293e3789a3.png?wh=1920*1124)
> > > 
> > > 1. 从treeIndex中获取key hello的版本号
> > > 
> > > 2. 再以版本号作为boltdb的key，从boltdb中获取其value信息
> > >    
> > >    1
> 
> #### treeIndex
> 
> > **treeIndex只保存用户的key和相关版本号信息，用户key的value数据存储在boltdb中**
> > 
> > 从treeIndex模块中获取hello这个key对应的版本号信息，treeIndex模块基于B-tree快速查找此key，返回此key的对应的索引项keyIndex。**keyIndex**包含版本号等信息
> 
> #### Buffer
> 
> > 获取了版本号信息之后，就可从boltdb模块中获取用户的key-value信息
> > 
> > **并不是所有的请求都是从boltdb中获取数据**
> > 
> > > etcd出于一致性、性能考虑，在访问boltdb前，会先从内存读事务buffer中，**二分法**查找key是否在buffer里面，命中直接返回。
> 
> #### boltdb
> 
> > 若未命中，则向boltdb模块查询数据
> > 
> > **boltdb基于B+tree来组织用户的key-value数据，获取到bucket key对象后，通过boltdb的游标cursor快速在b+tree找到key hello对应的value值，返回给client**
> 
> 一个读请求结束

## 7.2 etcd的写请求.

> #### 整体架构
> 
> ![](https://static001.geekbang.org/resource/image/8b/72/8b6dfa84bf8291369ea1803387906c72.png)
> 
> ##### 写请求全貌
> 
> > a.client端通过负载均衡算法选择一个etcd节点，发起gRPC调用
> > 
> > b.etcd节点收到请求后经过**gPRC拦截器**、**Quota模块**后，进入**KVServer模块**；
> > 
> > c.KVServer模块向Raft模块提交了提案，提案内容为“大家好，请使用put方法执行一个key为hello，value为world的命令”;
> > 
> > d.此后提案通过RaftHTTP模块、经过集群多个节点持久化后，转态变成已提交；
> > 
> > e.etcdserver从Raft模块获取已提交的日志条目，传递给Apply模块，Apply模块通过MVCC模块执行提案内容，更新状态机；
> > 
> > **备注：**
> > 
> > 1），写流程还涉及Quota、WAL、Apply模块
> > 
> > 2），crash-safe及幂等性也正是基于WAL和APPLY流程的consistent index等实现的
> 
> ### Quota模块
> 
> > 当etcd server收到put/txn等写请求时候，会首先检查当前etcd db加上你请求的key-value大小之和是否超过配额（quota-backend-bytes）
> > 
> > 如果超过了配额，他会产生一个警告（Alarm）请求，警告类型是NO SPACE，并通过Raft日志通过给其他节点，告知db无空间了，并将警告持久化存储到db中
> > 
> > 最终，无论API层的gRPC模块还是复制Raft侧已提交的日志条目应用到状态机的Apply模块，都是拒绝写入，集群只读；
> > 
> > > **为什么把配额（quota-backend-bytes）调大后，集群依然拒绝写入呢?**
> > > 
> > > 原因就是我们前面提到的 NO SPACE 告警：
> > > 
> > > > 1，Apply 模块在执行每个命令的时候，都会去检查当前是否存在 NO SPACE 告警，如果有则拒绝写入。所以还需要你额外发送一个取消告警**etcdctl alarm disarm**的命令，以消除所有告警；
> > > > 
> > > > 2，检查etcd的压缩（compact）配置是否开启、配置是否合理
> > > 
> > > 回收旧版本
> > > 
> > > > 1,压缩（compact）,回收旧版本，**它仅仅将旧版本占用的空间打上free标记，后续写入会复用这块空间，无需 申请新的空间**
> > > > 
> > > > 2，碎片整理（defrag)，回收空间，减小db大小，他会遍历旧的db文件数据，写入到一个新的db文件中去。**对集群性能影响较大**
> 
> ### KVServer模块
> 
> 通过流程二的配额检查后，请求就从 API 层转发到了流程三的 KVServer 模块的 put 方法，它需要将 put 写请求内容打包成一个提案消息，提交给 Raft 模块，在提案之前，还有如下一系列检查和限速。
> 
> > **preflight Check**
> > 
> > ![](https://static001.geekbang.org/resource/image/dc/54/dc8e373e06f2ab5f63a7948c4a6c8554.png)
> > 
> > > 1，为了保证集群稳定性，避免雪崩，任何提交到 Raft 模块的请求，都会做一些简单的限速判断；
> > > 
> > > 2，如果 Raft 模块已提交的日志索引（**committed index**）比已应用到状态机的日志索引（**applied index**）超过了 5000，那么它就返回一个"etcdserver: too many requests"错误给 client。
> > > 
> > > 3，它会尝试去获取请求中的鉴权信息，若使用了密码鉴权、请求中携带了 token，如果 token 无效，则返回"auth: invalid auth token"错误给 client
> > > 
> > > 4，它会检查你写入的包大小是否超过默认的 1.5MB， 如果超过了会返回"etcdserver: request is too large"错误给给 client。
> > 
> > **Propose**
> > 
> > > 1，通过一系列检查之后，会生产一个唯一ID，将此请求关联到一个对应的消息通知Channel，然后向Raft模块发起一个（Propose）一个提案（Propose）,提案内容为“大家好，请使用 put 方法执行一个 key 为 hello，value 为 world 的命令”；
> > > 
> > > 2，向Raft模块发起提案后，KVServer模块会等待此put请求，等待写入结果**通过消息通知Channel返回**或者超时。etcd默认超时时间是7秒（**5 秒磁盘 IO 延时 +2*1 秒竞选超时时间**），如果请求超时未返回，会出现**etcdserver: request timed out** 错误
> 
> ### WAL模块
> 
> > rafte模块收到提案后，如果当前节点是follower，他会转发给leader，**只有Leader才能处理写请求**。
> > 
> > Leader收到提案后，通过Raft模块**输出待转发给Follower节点的消息**和**待持久化的日志条目**，日志条目封装了我们上面说的put hello提案内容。
> > 
> > etcdserver从Raft模块获取到以上消息和日志条目后，作为Leader，他会将put提案广播给集群各个节点，同时需要把集群**Leader任期号**、**投票信息**、**已提交索引**、**提案内容**持久化到WAL（Write Ahead Log）日志中，用于保证集群一致性、可恢复性
> > 
> > > WAL日志结构
> > > 
> > > ![](https://static001.geekbang.org/resource/image/47/8d/479dec62ed1c31918a7c6cab8e6aa18d.png)
> > > 
> > > WAL 结构，它由**多种类型的 WAL 记录**顺序追加写入组成，每个记录由类型、数据、循环冗余校验码组成。不同类型的记录通过** Type 字段**区分，**Data 为对应记录内容**，**CRC 为循环校验码信息**
> > > 
> > > WAL 记录类型目前支持 5 种，分别是**文件元数据记录**、**日志条目记录**、**状态信息记录**、**CRC 记录**、**快照记录**
> > > 
> > > > 1，**文件元数据记录**包含节点 ID、集群 ID 信息，它在 WAL 文件创建的时候写入；
> > > > 
> > > > 2，**日志条目记录**包含 Raft 日志信息，如 put 提案内容；
> > > > 
> > > > 3，**状态信息**记录，包含集群的任期号、节点投票信息等，一个日志文件中会有多条，以最后的记录为准；
> > > > 
> > > > 4，**CRC 记录**包含上一个 WAL 文件的最后的 CRC（循环冗余校验码）信息， 在创建、切割 WAL 文件时，作为第一条记录写入到新的 WAL 文件， 用于校验数据文件的完整性、准确性等；
> > > > 
> > > > 5，**快照记录**包含快照的任期号、日志索引信息，用于检查快照文件的准确性。
> > > 
> > > WAL 模块持久化一个 put 提案的日志条目类型记录方式，put 写请求如何封装在 **Raft 日志条目**里面
> > > 
> > > > 1，Term 是 Leader 任期号，随着 Leader 选举增加；
> > > > 
> > > > 2，Index 是日志条目的索引，单调递增增加；
> > > > 
> > > > 3，Type 是日志类型，比如是普通的命令日志（EntryNormal）还是集群配置变更日志（EntryConfChange）；
> > > > 
> > > > 4，Data 保存我们上面描述的 put 提案内容。
> > > > 
> > > > ```
> > > > type Entry struct {
> > > >  Term uint64 `protobuf:"varint，2，opt，name=Term" json:"Term"`
> > > >  Index uint64 `protobuf:"varint，3，opt，name=Index" json:"Index"`
> > > >  Type EntryType `protobuf:"varint，1，opt，name=Type，enum=Raftpb.EntryType" json:"Type"`
> > > >  Data []byte `protobuf:"bytes，4，opt，name=Data" json:"Data，omitempty"`
> > > > }
> > > > ```
> > > > 
> > > > WAL 模块如何持久化 Raft 日志条目
> > > > 
> > > > > 1，首先先将 Raft 日志条目内容（含任期号、索引、提案内容）序列化后保存到 WAL 记录的 Data 字段；
> > > > > 
> > > > > 2，计算 Data 的 CRC 值，设置 Type 为 Entry Type， 以上信息就组成了一个完整的 WAL 记录。
> > > > > 
> > > > > 3，最后计算 WAL 记录的长度，顺序先写入 WAL 长度（Len Field），然后写入记录内容，调用 fsync 持久化到磁盘，完成将日志条目保存到持久化存储中
> 
> ### Apply模块
> 
> > ![](https://static001.geekbang.org/resource/image/7f/5b/7f13edaf28yy7a6698e647104771235b.png)
> > 
> > 核心就是我们上面介绍的 WAL 日志，因为**提交给 Apply 模块执行的提案已获得多数节点确认、持久化，etcd 重启时，会从 WAL 中解析出 Raft 日志条目内容，追加到 Raft 日志的存储中，并重放已提交的日志提案给 Apply 模块执行。**
> > 
> > > **问题一，如何确保幂等性，防止提案重复执行导致数据混乱呢?**
> > > 
> > > etcd 必须要确保幂等性。怎么做呢？
> > > 
> > > > 我们上面介绍 Raft 日志条目中的索引（index）
> > > > 字段。**日志条目索引是全局单调递增的**，**每个日志条目索引对应一个提案**， 如果一个命令执行后，我们在 db 里面也记录下当前已经执行过的日志条目索引，
> > > 
> > > **问题二：如果执行命令的请求更新成功了，更新 index 的请求却失败了，是不是一样会导致异常？**
> > > 
> > > > 我们在实现上，还需要将两个操作作为原子性事务提交，才能实现幂等
> > > > 
> > > > > etcd 通过引入一个 consistent index 的字段，来存储**系统当前已经执行过的日志条目索引**，实现幂等性
> > > 
> > > Apply 模块在执行提案内容前，首先会判断当前提案是否已经执行过了，如果执行了则直接返回，若未执行同时无 db 配额满告警，则进入到 MVCC 模块，开始与持久化存储模块打交道。
> 
> ### MVCC模块
> 
> Apply 模块判断此提案未执行后，就会调用 MVCC 模块来执行提案内容，
> 
> MVCC 主要由两部分组成：
> 
> > 1，一个是内存索引模块 treeIndex，**保存 key 的历史版本号信息**；
> > 
> > 2，另一个是 boltdb 模块，**用来持久化存储 key-value 数据**。
> > 
> > ![](https://static001.geekbang.org/resource/image/a1/ff/a19a06d8f4cc5e488a114090d84116ff.png)
> 
> **treeIndex**
> 
> > 问题：**当收到更新 key hello 为 world 的时候，此 key 的索引版本号信息是怎么生成的呢？需要维护、持久化存储一个全局版本号吗？**
> > 
> > > 1，版本号（revision）在 etcd 里面发挥着重大作用，它是 etcd 的逻辑时钟。etcd 启动的时候默认版本号是 1，**随着你对 key 的增、删、改操作而全局单调递增**。boltdb 中的 key 就包含此信息，所以 etcd 并不需要再去持久化一个全局版本号，**我们只需要在启动的时候，从最小值 1 开始枚举到最大值，未读到数据的时候则结束，最后读出来的版本号即是当前 etcd 的最大版本号 currentRevision**。
> > > 
> > > 2，MVCC 写事务在执行 put hello 为 world 的请求时，会基于 currentRevision 自增生成新的 revision 如{2,0}，然后从 treeIndex 模块中查询 key 的创建版本号、修改次数信息
> > > 
> > > 3，这些信息将填充到 boltdb 的 value 中，同时将用户的 hello key 和 revision 等信息存储到 B-tree，也就是下面简易写事务图的流程一，整体架构图中的流程八。
> 
> **boltdb**
> 
> > MVCC 写事务自增全局版本号后生成的 revision{2,0}，**它就是 boltdb 的 key**，通过它就可以往 boltdb 写数据了，进入了整体架构图中的流程九。
> > 
> > boltdb是一个基于 B+tree 实现的 key-value 嵌入式 db，它**通过提供桶（bucket）机制实现类似 MySQL 表的逻辑隔离**；
> > 
> > > 1，在 etcd 里面你通过 put/txn 等 KV API 操作的数据，全部保存在一个**名为 key 的桶**里面，**这个 key 桶在启动 etcd 的时候会自动创建**。
> > > 
> > > 2，保存用户 KV 数据的 key 桶，**etcd 本身及其它功能需要持久化存储的话，都会创建对应的桶**，上面我们提到的 etcd 为了保证日志的幂等性，保存了一个名为 consistent index 的变量在 db 里面，它实际上就**存储在元数据（meta）桶里面**；
> > 
> > 写入 boltdb 的 value 含有哪些信息呢？
> > 
> > > 写入 boltdb 的 value， **并不是简单的"world"**，**如果只存一个用户 value**，**索引又是保存在易失的内存上**，那**重启 etcd** 后，**我们就丢失了用户的 key 名，无法构建 treeIndex 模块了**。
> > > 
> > > 为了**构建索引**和**支持 Lease**等特性，etcd 会持久化以下信息:
> > > 
> > > > - key 名称；
> > > > 
> > > > - key 创建时的版本号（create_revision），最后一次修改时的版本号（mod_revision），key 自身修改的次数（version）；
> > > > 
> > > > - value 值；
> > > > 
> > > > - 租约信息（后面介绍）。
> > > 
> > > boltdb value 的值就是将含以上信息的结构体序列化成的二进制数据，然后通过 boltdb 提供的 put 接口，etcd 就快速完成了将你的数据写入 boltdb；
> > 
> > put 调用成功，就能够代表数据已经持久化到 db 文件了吗？
> > 
> > > 1，etcd 并未提交事务（commit），因此数据只更新在 boltdb 所管理的内存数据结构中；
> > > 
> > > 2，事务提交的过程，包含 B+tree 的平衡、分裂，将 boltdb 的脏数据（dirty page）、元数据信息刷新到磁盘，因此事务提交的开销是昂贵的；
> > > 
> > > etcd 的解决方案是合并再合并
> > > 
> > > > 1，首先 boltdb key 是版本号，put/delete 操作时，都会基于当前版本号递增生成新的版本号，因此属于顺序写入，可以调整 boltdb 的 bucket.FillPercent 参数，使每个 page 填充更多数据，减少 page 的分裂次数并降低 db 空间
> > > > 
> > > > 2，etcd 通过合并多个写事务请求，通常情况下，是异步机制定时（默认每隔 100ms）将批量事务一次性提交（pending 事务过多才会触发同步提交）， 从而大大提高吞吐量，对应上面简易写事务图的流程三
> > > 
> > > ？？？这优化又引发了另外的一个问题， 因为**事务未提交，读请求可能无法从 boltdb 获取到最新数据**；
> > > 
> > > > 1，etcd 引入了一个 bucket buffer 来保存暂未提交的事务数据。
> > > > 
> > > > 2，在更新 boltdb 的时候，etcd 也会同步数据到 bucket buffer。
> > > > 
> > > > 3，因此 etcd 处理读请求的时候会优先从 bucket buffer 里面读取，其次再从 boltdb 读，通过 bucket buffer 实现读写性能提升，同时保证数据一致性；

## 8，项目

## 9，kubernetes相关

### 9.1 kubernetes 的接口化和插件化

> CRI、CNI、CSI、CRD、Aggregated APIServer、Initializer、Device Plugin

### 9.1，~~kubernetes ~~相关开发

#### 9.1.1 CSI
