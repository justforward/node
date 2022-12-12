

# service


k8s之所以需要service，一方面是pod的ip是不固定的，另一个方面pod实例之间总有负载均衡的需求。

一个典型的service定义如下：
```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

这个service的例子，我们使用selector字段来声明这个service只代理携带了app=hostnames标签的pod，并且这个service的80端口，代理的是pod的9376端口。

然后，我们应用的Deployment，如图所示：
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: hostnames
spec:
 selector:
  matchLabels:
    app: hostnames
 replicas: 3
 template:
  metadata:
    labels:
      app: hostnames
  spec:
   containers:
   - name: hostnames
     image: k8s.gcr.io/serve_hostname
     ports:
     - containerPort: 9376
       protocol: TCP
```

这个应用的作用是每次访问9376端口的时候，返回他自己的hostname

**实验**

	step1: 创建Service和Deployment

```
# 创建Deployment
ginkgo :: ~/config/service % k create -f deployment.yaml
ginkgo :: ~/config/service % k get pod                                                   
NAME                        READY   STATUS    RESTARTS   AGE
hostnames-f69bc5549-fsnwq   1/1     Running   0          3h19m
hostnames-f69bc5549-jqqgb   1/1     Running   0          3h19m
hostnames-f69bc5549-vfjxz   1/1     Running   0          3h19m
# 创建service
ginkgo :: ~/config/service % k create -f service.yaml
ginkgo :: ~/config/service % k get endpoints hostnames
NAME        ENDPOINTS                                         AGE
hostnames   10.95.1.17:9376,10.95.2.11:9376,10.95.2.12:9376   3h26m
```

需要注意的是，只有处于Running状态，且readingessProbe检查通过的pod，才会出现在endpoints列表中，并且，当某个pod出现问题的时候，k8s会自动把他从service里面摘除掉。

	step2：查看service VIP

```
ginkgo :: ~/config/service % k get svc hostnames
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.96.61.215   <none>        80/TCP    3h30m

```

这个VIP是k8s自动为service分配的。

	step3： 通过service VIP 请求pod
	
```
ginkgo :: ~/config/service % curl 10.96.61.215:80
hostnames-f69bc5549-fsnwq
ginkgo :: ~/config/service % curl 10.96.61.215:80
hostnames-f69bc5549-vfjxz
ginkgo :: ~/config/service % curl 10.96.61.215:80
hostnames-f69bc5549-jqqgb
ginkgo :: ~/config/service % curl 10.96.61.215:80
```

通过三次连续不断的访问service的VIP地址和代理的端口80，他就为我们依次返回了三个pod的hostname，这个也证明了Serivce提供的是Round Robin方式的负载均衡，对于这种方式，我们成为ClusterIP模式的Service

## 原理

实际上，Service是由Kube-proxy组件，加上iptables来共同完成的。



# service 分类

![[Pasted image 20221208180306.png]]


## service 服务类型

Kubernetes service有四种类型——ClusterIP、NodePort、LoadBalancer 和 ExternalName。 service type 属性决定了服务如何暴露给网络。

### 1. ClusterIP(集群IP)

-   ClusterIP 是*默认*和最常见的服务类型。
-   Kubernetes 会为 ClusterIP 服务分配一个集群内部 IP 地址。 这使得服务只能在集群内访问。
-   您不能从集群外部向服务（pods）发出请求。
-   您可以选择在服务定义文件中设置集群 IP。


### 使用场景

集群内的服务间通信。 例如，应用程序的前端(front-end)和后端(back-end)组件之间的通信。

###  2.NodePort(节点端口)

-   NodePort 服务是 ClusterIP 服务的扩展。 NodePort服务路由到的 ClusterIP 服务会自动创建。
-   它通过在 ClusterIP 之上添加一个集群范围的端口来公开集群外部的服务。
-   NodePort 在静态端口（NodePort）上公开每个节点 IP 上的服务。每个节点将该端口代理到您的服务中。因此，*外部流量可以访问每个节点上的固定端口*。这意味着对该端口上的集群的任何请求都会转发到该服务。
		外部流量可以访问到每个节点上的固定端口
-   您可以通过请求 <NodeIP>:<NodePort> 从集群外部联系 NodePort 服务。
-   节点端口必须在 30000-32767 范围内。手动为服务分配端口是可选的。如果未定义，Kubernetes 会自动分配一个。
-   如果您要明确选择节点端口，请确保该端口尚未被其他服务使用。

#### 使用场景

-   当您想要启用与您的服务的外部连接时。
-   使用 NodePort 可以让您自由地设置自己的负载平衡解决方案，配置 Kubernetes 不完全支持的环境，甚至直接公开一个或多个节点的 IP。
-   最好在节点上方放置负载均衡器以避免节点故障。




# kube-proxy

Kube-proxy 是 kubernetes 工作节点上的一个网络代理组件，运行在每个节点上。

kube-proxy 维护节点上的网络规则，实现了kubernrtes Service概念的一部分。它的作用是使发往service的流量（通过clusterIP和端口）负责均衡到正确的后端pod

## 工作原理

kube-proxy 监听 API server 中 资源对象的变化情况，包括以下三种：

-   service
-   endpoint/endpointslices
-   node

然后根据监听资源变化操作代理后端来为服务配置负载均衡。

## endpoint vs endpointslices

原来的 Endpoints API 提供了在 Kubernetes 中跟踪网络端点的一种简单而直接的方法。随着 Kubernetes 集群和[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)逐渐开始为更多的后端 Pod 处理和发送请求， 原来的 API 的局限性变得越来越明显。最明显的是那些因为要处理大量网络端点而带来的挑战。


使用 EndpointSlices 时，添加或移除单个 Pod 对于正监视变更的客户端会触发相同数量的更新， 但这些更新消息的大小在大规模场景下要小得多。

使用EndpointSlices时候，添加或者移除单个pod对于整

## 代理模式

目前 Kube-proxy 支持4中代理模式：

-   userspace（wind上的支持）
-   iptables
-   ipvs
-   kernelspace

### iptables

![[Pasted image 20221208173317.png]]


默认的策略是，kube-proxy 在 iptables 模式下随机选择一个后端。

如果 kube-proxy 在 iptables 模式下运行，并且所选的第一个 Pod 没有响应， 则连接失败。 这与用户空间模式不同：在这种情况下，kube-proxy 将检测到与第一个 Pod 的连接已失败， 并会自动使用其他后端 Pod 重试。

但是，kube-proxy对iptables规则进行编程的方式是一种O(n)复杂度的算法，其中n与集群大小(或更确切地说，服务的数量和每个服务背后的后端Pod的数量）成比例地增长)。


### IPVS

IPVS是专门用于负载均衡的Linux内核功能。在IPVS模式下，kube-proxy可以对IPVS负载均衡器进行编程，而不是使用iptables。这非常有效，它还使用了成熟的内核功能，并且IPVS旨在均衡许多服务的负载。它具有优化的API和优化的查找例程，而不是一系列顺序规则。 结果是IPVS模式下kube-proxy的连接处理的计算复杂度为O(1)。换句话说，在大多数情况下，其连接处理性能将保持恒定，而与集群大小无关。

![[Pasted image 20221208173549.png]]


IPVS的一个潜在缺点是，与正常情况下的数据包相比，由IPVS处理的数据包通过iptables筛选器钩子的路径不同。如果打算将IPVS与其他使用iptables的程序一起使用，则需要研究它们是否可以一起正常工作。 不过Ipvs代理模式已经推出很久了，很多组件已经适配的很好了，比如Calico。


### 性能对比总结

对于iptables和IPVS模式，kube-proxy的响应时间开销与建立连接相关，而不是与在这些连接上发送的数据包或请求的数量有关。这是因为Linux使用的连接跟踪（conntrack）能够非常有效地将数据包与现有连接进行匹配。如果数据包在conntrack中匹配，则无需检查kube-proxy的iptables或IPVS规则即可确定该如何处理。

在集群中不超过1000个服务的时候，iptables 和 ipvs 并无太大的差异。而且由于iptables 与网络策略实现的良好兼容性，iptables 是个非常好的选择。

当你的集群服务超过1000个时，而且服务之间链接大多没有开启keepalive，IPVS模式可能是一个不错的选择。



