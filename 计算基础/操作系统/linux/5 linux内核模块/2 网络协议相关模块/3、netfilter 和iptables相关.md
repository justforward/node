
在网络层中起到作用。

## 什么是netfilter和iptables

-   netfilter指整个项目，不然官网就不会叫www.netfilter.org了。
-   在这个项目里面，netfilter特指内核中的netfilter框架，iptables指用户空间的配置工具。
-   netfilter在协议栈中添加了5个钩子，允许内核模块在这些钩子的地方注册回调函数，这样经过钩子的所有数据包都会被注册在相应钩子上的函数所处理，包括修改数据包内容、给数据包打标记或者丢掉数据包等。
-   netfilter框架负责维护钩子上注册的处理函数或者模块，以及它们的优先级。
-   iptables是用户空间的一个程序，通过netlink和内核的netfilter框架打交道，负责往钩子上配置回调函数。
-   netfilter框架负责在需要的时候动态加载其它的内核模块，比如 ip_conntrack、nf_conntrack、NAT subsystem等。


> 在应用者的眼里，可能iptables代表了整个项目，代表了防火墙，但在开发者眼里，可能netfilter更能代表这个项目。


## netfilter钩子（hooks）

```
         |
         | Incoming
         ↓
+-------------------+
| NF_IP_PRE_ROUTING |
+-------------------+
         |
         |
         ↓
+------------------+
|                  |         +----------------+
| routing decision |-------->| NF_IP_LOCAL_IN |
|                  |         +----------------+
+------------------+                 |
         |                           |
         |                           ↓
         |                  +-----------------+
         |                  | local processes |
         |                  +-----------------+
         |                           |
         |                           |
         ↓                           ↓
 +---------------+          +-----------------+
 | NF_IP_FORWARD |          | NF_IP_LOCAL_OUT |
 +---------------+          +-----------------+ // 本地程序要发出去的数据包刚到IP层，
												// 还没进行路由选择
         |                           |
         |                           |
         ↓                           |
+------------------+                 |
|                  |                 |
| routing decision |<----------------+
|                  |
+------------------+
         |
         |
         ↓
+--------------------+
| NF_IP_POST_ROUTING |
+--------------------+    // 本地程序发出去的数据包，或者转发(forward)的数据包已经经过了
                          // 路由选择，即将交由下层发送出去
         |
         | Outgoing
         ↓
```

-   NF_IP_PRE_ROUTING: 接收的数据包刚进来，还没有经过路由选择，即还不知道数据包是要发给本机还是其它机器。
    
-   NF_IP_LOCAL_IN: 已经经过路由选择，并且该数据包的目的IP是本机，进入本地数据包处理流程。
    
-   NF_IP_FORWARD: 已经经过路由选择，但该数据包的目的IP不是本机，而是其它机器，进入forward流程。
    
-   NF_IP_LOCAL_OUT: 本地程序要发出去的数据包刚到IP层，还没进行路由选择。
    
-   NF_IP_POST_ROUTING: 本地程序发出去的数据包，或者转发（forward）的数据包已经经过了路由选择，即将交由下层发送出去。

从上面的流程中，我们还可以看出，不考虑特殊情况的话，一个数据包只会经过下面三个路径中的一个：

-   本机收到目的IP是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN
    
-   本机收到目的IP不是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_FORWARD -> NF_IP_POST_ROUTING
    
-   本机发出去的数据包: NF_IP_LOCAL_OUT -> NF_IP_POST_ROUTING


注意： netfilter所有的钩子（hooks）都是在内核协议栈的IP层，由于IPv4和IPv6用的是不同的IP层代码，所以iptables配置的rules只会影响IPv4的数据包，而IPv6相关的配置需要使用ip6tables。

## iptables中的表（tables）

iptables用表（table）来分类管理它的规则（rule），根据rule的作用分成了好几个表，比如用来过滤数据包的rule就会放到filter表中，用于处理地址转换的rule就会放到nat表中，其中rule就是应用在netfilter钩子上的函数，用来修改数据包的内容或过滤数据包。目前iptables支持的表有下面这些：

#### Filter

从名字就可以看出，这个表里面的rule主要用来过滤数据，用来控制让哪些数据可以通过，哪些数据不能通过，它是最常用的表。

#### NAT

里面的rule都是用来处理网络地址转换的，控制要不要进行地址转换，以及怎样修改源地址或目的地址，从而影响数据包的路由，达到连通的目的，这是家用路由器必备的功能。

#### Mangle

里面的rule主要用来修改IP数据包头，比如修改TTL值，同时也用于给数据包添加一些标记，从而便于后续其它模块对数据包进行处理（这里的添加标记是指往内核skb结构中添加标记，而不是往真正的IP数据包上加东西）。

#### Raw

在netfilter里面有一个叫做connection tracking的功能（后面会介绍到），主要用来追踪所有的连接，而raw表里的rule的功能是给数据包打标记，从而控制哪些数据包不被connection tracking所追踪。

#### Security

里面的rule跟SELinux有关，主要是在数据包上设置一些SELinux的标记，便于跟SELinux相关的模块来处理该数据包。







## 扩展 

### 什么是netlink

Netlink[套接字](https://so.csdn.net/so/search?q=%E5%A5%97%E6%8E%A5%E5%AD%97&spm=1001.2101.3001.7020)是用以实现**用户进程**与**内核进程**通信的一种特殊的进程间通信(IPC) ,也是网络应用程序与内核通信的最常用的接口。

