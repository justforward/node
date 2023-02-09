-   netfilter指整个项目，不然官网就不会叫www.netfilter.org了。
-   在这个项目里面，netfilter特指内核中的netfilter框架，iptables指用户空间的配置工具。
-   netfilter在协议栈中添加了5个钩子，允许内核模块在这些钩子的地方注册回调函数，这样经过钩子的所有数据包都会被注册在相应钩子上的函数所处理，包括修改数据包内容、给数据包打标记或者丢掉数据包等。



https://segmentfault.com/a/1190000009043962?utm_source=sf-backlinks



netfilter特指内核中netfilter框架，iptables指用户空间的配置工具。