https://info.support.huawei.com/info-finder/encyclopedia/zh/NAT.html



随着网络应用的增多，IPv4地址枯竭的问题越来越严重。尽管[IPv6](https://info.support.huawei.com/info-finder/encyclopedia/zh/IPv6.html "IPv6")可以从根本上解决IPv4地址空间不足问题，但目前众多网络设备和网络应用大多是基于IPv4的，因此在IPv6广泛应用之前，使用一些过渡技术（如CIDR、私网地址等）是解决这个问题的主要方式，NAT就是这众多过渡技术中的一种。

当私网用户访问公网的报文到达网关设备后，如果网关设备上部署了NAT功能，设备会将收到的IP数据报文头中的IP地址转换为另一个IP地址，端口号转换为另一个端口号之后转发给公网。在这个过程中，设备可以用同一个公网地址来转换多个私网用户发过来的报文，并通过端口号来区分不同的私网用户，从而达到地址复用的目的。

