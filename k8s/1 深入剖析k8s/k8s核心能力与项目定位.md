![[Pasted image 20221204214057.png]]

k8s项目的全局架构


分为master节点和Node节点，

## master节点


1）master节点为控制节点，由3个紧密协作的独立组件组合而成，分别是负责API服务的kube-apiserver、负责调度的kube-scheduler（调度器）、以及负责容器编排服务的kube-controller-manager（控制管理器）。
整个集群的持久化数据，则由kube-apiserver处理后保存在etcd中

## Node 节点

2）node节点为计算节点，最核心的部分是一个名为kubectl的组件，它主要负责和容器运行时(比如docker项目)交互，而这种交互所依赖的是一个称为CRI（container runtime interface， 容器运行时接口）的远程调用接口，该接口定义了容器运行时的各项核心操作，比如启动一个容器所需要的所有参数，runc--运行时引擎。

## 容器运行时交互

### CRI

kubelet 调用下层容器运行时的执行过程，并不会直接调用Docker 的 API，而是通过一组叫作 CRI（Container Runtime Interface，容器运行时接口）的 gRPC 接口来间接执行的。Kubernetes 项目之所以要在 kubelet 中引入这样一层单独的抽象，当然是为了对 Kubernetes 屏蔽下层容器运行时的差异。
CRI的结构如下：

![[Pasted image 20221204220459.png]]

kubelet 通过grpc框架与CRI shim进行通信，CRI shim通过Unix socket 启动一个grpc server 提供容器运行时服务。grpc service使用protobuffer 提供两类GRPC service：ImageService和RuntimeService。Imageservice 提供从镜像仓库中拉取、删除、查询镜像信息的RPC调用。RuntimeService 提供容器的相关生命周期管理以及容器的交互操作。

*接口与实现*
 CRI最核心的两个概念是PodSandbox和container，pod由一组应用容器组成，这些应用容器共享相同的环境和资源约束，这个共同的环境与资源约束被称为podsandbox，由于不同的容器运行时对podsand实现不一样，所以CRI留有一组接口给不同的容器运行时自主发挥，比如hypervisor将podSandbox实现成一个虚拟机，docker则将podsandbox实现成一个linux命名空间。

*Runtimeservice*

kubectl在创建一个pod之前需要调用RuntimeService。RunPodSandbox 为pod创建一个podSandbox，这个过程包含初始哈pod网络，分配IP 激活沙箱等。然后，kubectl会调用CreateContainer、StartContainer、StopContainer、RemoveContainer对容器进行创建、启动、暂停和删除等操作，等待pod被删除时候，会调用stopPodSandbox，RemovePodSandbox来销毁pod。kubectl的职责在于pod的生命周期的管理，包含监控检测与重启策略控制，并且实现容器声明周期管理中的各种钩子。

*ImageService*

为了启动一个容器，CRI还需要执行镜像相关的操作，比如镜像的拉取，查看 移除。但是容器的运行时不需要镜像的构建操作，因此CRI中并不包括buildImage的操作，镜像的构建需要使用额外的工具Docker来完成。



这也是为何k8s项目不关心你部署的是什么容器运行时、使用了什么技术实现，只要你的容器运行时能够运行标准的容器镜像，它就可以通过实现CRI接入k8s项目，而具体的容器运行时，比如docker项目，则一般通过OCI容器规范，包括运行时规范和镜像规范（runc就是一个实现）同底层的linux操作系统进行交互，即把CRI请求翻译成对linux操作系统的调用。


**容器网络的区别**

在正常情况下，pod中容器会共享一个Network Namespace，因此pod需要在创建Sandbox容器

### 网络插件

作为容器配置网络 CNI

pod网络初始化的流程：
	1、kubelet调用CRI runtime（dockershim或者containerd）创建pod的Sandbox容器
	2、CRI runtime调用底层docker或者containerd创建pause容器（此时pause进程还没启动，但是已经初始化network namespace）
	3、CRI Runtime调用CNI执行在network Namespace中创建veth并加入cbr0网桥等网络操作
	4、CRI Runtime 启动pause容器，pause进程被拉起
	5、kubectl 继续调用CRI Runtime执行创建容器等后续操作
	
Docker 和 containerd 在**创建 pause 容器并初始化 Network Namespace**和 **调用 CNI 初始化 veth** 这两步有区别。

*创建pause容器*

Containerd是为了k8s设计的CRI runtime，没有独立的网络模块，但dokcer在设计时候带有自己的网络功能，因此docker在创建pause容器时候，会进行docker特有的网络设置，该设置导致和containerd最大区别是在不使用IPV6的情况下，Docker 会将容器 Network Namespace 中内核参数`net.ipv6.conf.all.disable_ipv6` 设置为1，也即关闭容器内的 ipv6 选项。

**同样的 Pod 在 Docker 的节点上只开启了 IPv4，而在 Containerd 的节点上会同时开启 IPv4 和 IPv6。**

同时开启 IPv4 和 IPv6 的情况中，DNS 解析可能会同时发出v4、v6两个版本的包。在某些情况，业务如果需要频繁进行 DNS 解析，可能会触发 DNS 解析库的 Bug（取决于 Pod 业务的实现时的依赖）。在 Containerd 节点上，可以通过给 Pod 添加 init container 来针对 Pod 关闭 IPv6 设置。代码如下：

```
apiVersion: v1
kind: Pod
...
spec:
  initContainers:
  - image: busybox
    name: sysctl
    command: ["sysctl", "-w", "net.ipv6.conf.all.disable_ipv6=1"]
    securityContext:
      privileged: true
...
```



### 存储插件

容器持久化存储 CSI
