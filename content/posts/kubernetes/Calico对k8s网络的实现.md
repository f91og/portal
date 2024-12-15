+++
title = 'Calico 对 k8s 网络的实现'
date = 2024-12-15T07:54:27Z
draft = false
tags = ['network','难点分析','calico']
categories = ['Kubernetes']
+++

Calico 是 k8s 常用的 cni 插件之一，部署简单而且性能还不错，主要实现的是 pod ip 分配， pod 到 pod 的通信，以及网络策略等。

<!--more-->
## 理解网络通信的几个要点
在理解 calico 的网络结构之前，先搞明白几个以前困惑的问题。

**I. ip 地址和 mac 地址在数据包传送时的区别**  
如果靠 ip 地址就能确定通信的话那么还要 mac 地址干什么咧？答案就是 ip 地址是个网络层上的东西，靠这个只能将数据包送达到目的地的所在的网络中，最后还是要靠 mac 地址来确定目的主机，交换机根据这个 mac 地址将数据包发送给它。
大致的流程：
1. 在网络 A 里，arp 广播得到 ip 地址的 mac 地址。
2. 在网络 A 中，数据包从某个主机被发出，这时候有目的 ip，但肯定是不知道目的地 mac 地址的，然后数据包被送到了这个网络里的交换机（机器直连交换机，机器不会连着机器的），交换机通过 arp 广播问网络中哪个机器有目的 ip，并返回它的 mac 地址。
3. 如果交换机在本网络中找到了，那么就通过 mac 地址将数据包送达给目的地，交换机是工作在第二层的，只知道 mac 地址。
4. 如果本网络中没有找到，则把数据包给默认网关，一般是路由器，路由器的路由表中记载了到达目的 ip 的下一跳 ip 是什么，把数据包给下一跳。
5. 层层跳转后，数据包终于来到了目的主机所在的网络，此时要通过交换机将这个数据包给目的主机，所以交换机还是要知道 mac 地址，如果交换机本地的 mac 地址表没有的话，发 arp 广播让目的主机告诉它的 mac 地址，最终将数据包给到目的主机。

总结：数据包的传送最终底层还是依赖于 mac 地址，依赖于二层设备交换机。ip 地址工作在网络层，主要作用对象是工作在第三层的路由器，用来寻找目的网络，只不过听上去像是定位主机位置的感觉，其实也算是定位了吧，只不过是在网络层这个抽象的概念上的定位，类似于一个家的门牌号地址和实际的地理位置（地图上的经纬度）的区别，最终快递员还是通过地图上房子的实际位置来派送物件。

**II. 集群里的主机到 public internet 的 http 连接后，数据包的来回是不是相同的路径**  
非常有可能，比如从数据中心到公网是通过过 NAT，而入站流量通过 LB，这种来回路径不同的情况也被称为**非对称路由**。
## calico 的架构
各种 CNI 插件本质做的事情都差不多，改改 iptable，route tables 网络设备啥的，所以在分析 Calico 的架构之前大概也能猜得出一些：
1. 每个 node 上有个进程负责维护 iptable 和网络设备，这个通过 k8s ds 可以做到。
2. 有个中心化的控制组件，用于管理配置，记录路由信息等
3. 可能还需要一个后端稳定的存储，如果光靠文件存储还是不太稳定，机器挂了就完蛋了。

Calico 架构图：
![](/images/20241214201341.png)
基本上和猜的差不多：
- calico-node 作为 ds 维护每个节点上的 iptable 和 route table
- calico-kube-controller 监控 k8s 资源的配置变化，作为 deployment 的 pod 部署在集群里，如果 network policy 发生变化要把这个变化推送给 calico-node 让其更改 iptable 和 route table
- Calico database 存储一些 Calico 自己的资源，不过这里应该可以通过配置将 etcd 指定为 Calico 的数据库
- typha 是可选组件，作为中间代理，减少多个 Calico 节点直接与 Kubernetes API 交互带来的负载，提高大规模集群中的同步效率

一个集群中的具体 calico 部署案例大概就是这样:
```shell
kg deploy -n kube-system | grep calico
calico-kube-controllers   1/1     1            1           324d
calico-typha              3/3     3            3           324d

kg ds -n kube-system | grep calico
calico-node                282       282       282     282          282         kubernetes.io/os=linux                        297d
```

其中 calico-node 中有 2 个组件，他们的作用是：
- **Felix**：Calico 的代理（Agent），负责配置 **iptables** 和 **路由表**，实现数据包的安全策略（Network Policy）和流量转发
- **BIRD**：BGP 路由协议的实现，负责将节点的网络路由信息分发到其他节点，实现数据包跨节点通信
## pod 到宿主机的网络
可以先看这个：[Calico 网络通信解析](https://www.ffutop.com/posts/2019-12-24-how-calico-works/)  
首先在 pod 里看路由表信息：
```shell
# ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link 
```
可以看到默认网关是 `169.254.1.1`，[169.254.1.1 保留IP地址 | IP地址 (简体中文) 🔍](https://zh-hans.ipshu.com/ipv4/169.254.1.1) 是一个特殊的地址，这里 calico 将这个地址设置为 pod 的默认网关主要是让这个 ip 不与其他 ip 地址冲突。

尝试在 pod 里 traceroute 一下：
```shell
# traceroute -p 443 dev.globalad-portal.dev.jp.local
traceroute to dev.globalad-portal.dev.jp.local (100.73.3.215), 30 hops max, 46 byte packets
 1  dev-node2196z.dev.jp.local (100.73.172.30)  0.005 ms  0.006 ms  0.004 ms
 2  stg-fw01-jpe2-vlan2464-v.dev.jp.local (100.73.172.254)  0.164 ms  0.201 ms  0.214 ms
 3  dev.globalad-portal.dev.jp.local (100.73.3.215)  0.255 ms  0.359 ms  0.277 ms
```
从结果来看从 pod 出发的请求下一跳是宿主机 `dev-node2196z.dev.jp.local (100.73.172.30)`，要搞明白的是为什么到网关 `169.254.1.1` 的请求可以到宿主机。

Calico 会通过 veth pair 将宿主机的网卡和容器中的网卡连接，[2.5.2 虚拟以太网设备 Veth | 深入架构原理与实践](https://www.thebyte.com.cn/content/chapter1/veth-pair.html)，首先查看宿主机的情况：
```shell
# ip link show type veth
5: calie1d6b5012fb@ens224: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
7: caliba9951e9d21@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
571: cali5475d974730@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
573: cali20a7f22e20a@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
574: cali4d7f87a2695@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
575: cali961628a8290@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
.......

# ip link show type bridge 
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether ***************** brd *****************
```

到 pod 里看下网络设备的情况：
```shell
# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback ***************** brd *****************
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1
    link/ipip ******* brd *******
4: eth0@if571: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether ***************** brd *****************
```
发现 eth0 的另一端是 `if571`，后面的 571 数字表示的是和宿主机的这个 calico 接口连在一起：
```shell
571: cali5475d974730@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ***************** brd *****************
```

当在 pod 中数据要被发到 `169.254.1.1` 时，先需要通过 arp 广播确定它的 mac 地址是什么，很明显在整个宿主机网络和容器网络里没有任何网卡的 ip 地址是 `169.254.1.1`，这个时候通过 calico 接口通过 `proxy_arp` 功能使其 ip 地址就算不是 `169.254.1.1` 也会响应 arp 广播并返回它 mac 地址。

确认下宿主机网卡的 `proxy_arp` 功能：
```shell
# cat /proc/sys/net/ipv4/conf/ens192/proxy_arp
0
# cat /proc/sys/net/ipv4/conf/ens224/proxy_arp
0
# cat /proc/sys/net/ipv4/conf/cali4449b2e925d/proxy_arp
1
# cat /proc/sys/net/ipv4/conf/cali5fff92488be/proxy_arp
1
# cat /proc/sys/net/ipv4/conf/cali5591465738e/proxy_arp
1
```
可以看到所有 calixxx 网卡都开启了 `proxy_arp`。
## 跨 node 时的网络流程
当 pod 去访问同集群或者不同集群中另一个 node 上 pod 的时候，因为要知道目标 node 的 ip 以及目标 pod 的 ip，基本实现方法有 2 种：
1. 包裹原数据包，从 pod 出来的数据包目的 ip 只有目的 pod 的 ip 所以要在这数据包中加入源 node 和目的 node 的 ip，接收端要做额外解析，有一定的资源消耗（Flannel 的做法）。
2. **直接路由实现（不需要包裹数据包）**，在这种方法中，**Pod 到 Pod 的通信通过直接路由的方式实现**，数据包不会被额外封装（不需要额外的 encapsulation），而是基于主机的路由表或者 BGP（边界网关协议）来完成路由。目标 Node 和目标 Pod 的 IP 都是可以直接路由的。

Calico 采用了 **第二种方法**，通过 **原生路由机制** 实现 Pod 到 Pod 的通信，主要有以下特点：
1. **BGP（边界网关协议）**：  
    每个节点（Node）运行一个 BGP 客户端（如 BIRD），通过 BGP 协议将 Pod 的网络前缀（子网范围）通告给整个集群的路由表。这使得每个 Node 都知道其他 Node 上 Pod 网络的路由路径。
2. **无需封装（No Encapsulation）**：  
    数据包直接基于路由表从源 Pod 发送到目标 Pod，不需要封装成 IPIP、VXLAN 等隧道形式。这避免了封装带来的资源消耗，提高了网络性能。
3. **路由表更新**：  
    当 Pod 网络发生变更（比如新 Pod 部署或删除）时，Calico 的组件（如 Felix 和 BGP Daemon）动态更新路由表，确保路由信息是最新的。
4. **多种数据存储选项**：  
    Calico 可以使用 Kubernetes API Server 作为配置数据存储，也可以使用 etcd 来存储网络策略和路由信息。
Calico 的直接路由实现网络性能更高，尤其在大规模集群场景下更为高效，但需要配置和维护路由信息（通常借助 BGP）。

可以在集群中验证下 Calico 的实现，先看下集群中 podA 的 ip 和 它的 node 的 ip:
```shell
busybox-6644b5988-6wsxw    192.168.161.72  dev-node2210z.dev.jp.local
dev-node2210z.dev.jp.local  100.73.172.44
```

在另一个节点 nodeB 上确认路由信息
```shell
# ip route | grep 192.168.161
192.168.161.0/26 via 100.73.172.35 dev tunl0  proto bird onlink 
192.168.161.64/26 via 100.73.172.44 dev tunl0  proto bird onlink 
192.168.161.128/26 via 100.73.97.44 dev tunl0  proto bird onlink 
```
根据网络范围判断可以知道 `192.168.161.72 ` 在 `192.168.161.64/26` 这个网段里，所以从 nodeB 上如果往 podA 发请求的话，就知道下一个路由目标是 `100.73.172.44`，即 podA 所在的 node 的 ip，再在 podA 的 node 上看路由信息：
```shell
# ip route | grep 192.168.161
192.168.161.0/26 via 100.73.172.35 dev tunl0  proto bird onlink 
blackhole 192.168.161.64/26  proto bird 
192.168.161.65 dev caliba9951e9d21  scope link 
192.168.161.72 dev cali5475d974730  scope link 
192.168.161.79 dev cali700f1e4f8eb  scope link 
192.168.161.82 dev cali61ec5cc43ec  scope link 
192.168.161.83 dev cali4d7f87a2695  scope link 
192.168.161.91 dev calib219a8a4cbc  scope link 
192.168.161.96 dev cali1e08058e9d8  scope link 
192.168.161.97 dev calie1d6b5012fb  scope link 
192.168.161.104 dev cali20a7f22e20a  scope link 
192.168.161.105 dev cali20fdce43e7b  scope link 
192.168.161.106 dev cali724c82f2335  scope link 
192.168.161.114 dev calice0461d0744  scope link 
192.168.161.119 dev cali961628a8290  scope link 
192.168.161.120 dev cali4edf734636c  scope link 
```
`192.168.161.72` 走网卡 `cali5475d974730`，它连着容器端的网卡，这样 pod 到 pod 的路由就通了。

上面是同集群不同 node 的情形，还有一种情形是跨集群的 node 和 node 之间的路由，这个和 BGP 有关，有点复杂，有时间的话再来具体分析下，但不管怎样就实现层面来说都是 Calico 来负责维护路由信息，让从 pod 出来的数据包可以正确到达目的地。
## 备注
尝试去另一个集群中看 calico 的情况，发现 calixxx 接口后面不是 docker0
```shell
11: cali5d4370e6c12@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8980 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ***************** brd ***************** link-netns cni-************************************
12: cali13c98071982@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8980 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ***************** brd ***************** link-netns cni-************************************
```
不知道是为啥，有时间的话再来研究研究下。