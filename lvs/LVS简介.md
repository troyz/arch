### 1，简介
LVS是Linux Virtual Server的简写，意即Linux虚拟服务器，是一个虚拟的服务器集群系统。本项目在1998年5月由章文嵩博士成立，是中国国内最早出现的自由软件项目之一。
Linux2.6后ipvs已经成为了Linux内核的一部分。
###2，LVS工作原理
![LVS工作原理](http://ww2.sinaimg.cn/large/6403b6a7jw1eoymzdbr9qj20g206dq32.jpg)

> ipvs工作在Linux内核进程中，从Linux2.6以后的版本都自带，所以我们只需要安装ipvsadm软件。
> 
> [LVS官网](http://www.linuxvirtualserver.org/), [中文站点](http://zh.linuxvirtualserver.org/)

###3，LVS工作模型
LVS有3种工作模型：
> NAT: 网络地址协调转换
> DR: 直接路由
> TUN: IP隧道

|   模型   |  特点  |
| :--: | ---- |
|  NAT  |  网络地址协调转换 |
|       |  Director的dip作为rip的网关工作, dip/rip工作在内网  |
|       | 所有的集群节点都必须在同一个网络，**不能跨网段** |
|       | 通常情况下，rip是私有地址，仅用于集群节点间通信 |
|       | director同时处理**入栈出栈**连接（来自客户端的请求和来自real server的响应数据包都要经过director） |
|       | **real server的网关要指向dip** |
|       | 可以实现端口映射 |
|       | real server可以是任意操作系统 |
|       | director很容易成为系统性能瓶颈 |
|  DR  | 直接路由 |
|       | director和real server必须在同一个物理网络，不能跨路由器，（因为是基于mac地址转发的） |
|       | rip可以使用公网ip，（万一director挂了，可以直接访问rip） |
|       | director仅处理**入栈**请求，而real server的响应数据包不再经过director，因此real server的网关一定不能指向director |
|       | director不支持端口映射 |
|       | 大多数的操作系统都支持real ip |
|       | director压力小，性能优。 |
|       | 在实际使用中，应该使用DR模型。 |
|  TUN  | ip隧道 |
|       | 与DR原理相似。 |
|       | director与real server可以不在同一个网络，可跨互联网。 |
