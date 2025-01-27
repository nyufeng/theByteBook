# 4.4 四层负载均衡技术

四层负载均衡器的典型代表是 LVS（Linux Virtual Server，Linux 虚拟服务器）。

从 Linux 内核 2.4 版本起，LVS 被集成进 Linux 内核，它基于本书第三章介绍的 Netfilter 的钩子捕获数据包并进一步处理，然后根据负载均衡算法转发至“后端服务器”。

:::tip 
LVS 称 Linux 虚拟服务器，相应的，实际执行请求的称真实服务器（real server）。为了简化表述，本节统一称“后端服务器”。
:::

区别数据包的处理方式，LVS 有了三种工作模式：直接路由（DR，Direct Routing）、隧道（Tunnel）和网络地址转换（NAT，Network Address Translation）。

## 4.4.1 直接路由模式

LVS 的直接路由模式，实际上是一种链路层负载均衡技术。

链路层负载均衡的原理是，负载均衡器（LVS）接收请求后，修改数据帧的目标 MAC 地址，将原本发给负载均衡器的数据帧重新定向，经过二层交换机转发至某个“后端服务器”。


后端服务器解包数据帧，会发现 IP 层的目的 IP 并非本机，所以 Linux 内核网络协议栈无法继续处理该数据包。为了解决这个问题，后端服务器需要将虚拟 IP（VIP，Virtual IP）配置到本地回环接口（lo）上。这样，Linux 内核网络协议栈就能识别并正常处理该数据包。

如某个 VIP 为 1.1.1.1，通过以下命令将该 IP 绑定到后端服务器的 lo 接口：

```bash
// 这里 /32 表示单个 IP 地址，也即仅将 VIP 绑定到 lo 接口，不与其他 IP 地址共享子网。
$ ip addr add 1.1.1.1/32 dev lo
```

为了避免后端服务器对 VIP 抢答 ARP 请求，从而导致网络问题，还必须禁用后端服务器对 VIP 的 ARP 应答。命令如下所示：

```bash
$ echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
$ echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce

// 为了确保 eth0 或其他网络接口也不会响应该 VIP 的 ARP 请求，需要对每个网络接口设置相同的 ARP 参数。
$ echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
$ echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

经过上述配置后，后端服务器当在本地回环接口（lo）上检测到目标 IP 为 VIP（配置的 1.1.1.1）的数据包时，将其识别为自己的地址并处理请求。处理完成后，后端服务器通过其外部接口（如 eth0）发送响应数据时，会保持原始请求中的目的 IP 地址，即 VIP（1.1.1.1），作为响应数据包的源 IP。

直接路由模式中，请求经过负载均衡器后，后端服务器的响应无需通过负载均衡器原路返回。整个工作模式中，请求、转发和响应形成了一个“三角关系”，因此直接路由模式也被形象地称为“三角传输模式”，如图 4-9 所示。

:::center
  ![](../assets/balancer4-dsr.svg)<br/>
 图 4-9 直接路由模式的三角传输示例
:::

直接路由模式的主要优势在于它特别适合于响应流量远大于请求流量的场景。如典型的 HTTP 请求/响应模式，假设请求流量仅占 10%，响应流量占 90%。通过三角传输模式，负载均衡器只需处理 1/10 的总流量，这样不仅极大节省了带宽成本，还提高了负载均衡器的可靠性（流量越低越好）。

当然，直接路由模式缺陷也非常明显：
- 首先，由于响应流量直接返回给客户端，中间的负载均衡器就无法监控完整的 TCP 连接状态，影响防火墙策略（负载均衡器只能看到 TCP 连接的 SYN 包，而无法看到 ACK 包）；
- 其次，负载均衡器与后端服务器之间通过链路层通信，它们必须位于同一子网内，对网络架构有一定的要求。

## 4.4.2 隧道模式

直接路由模式通过改写数据链路层的 MAC 地址实现请求转发。到了网络层，我们也可以采用类似的方式修改网络层的 IP 数据包，实现请求的转发。LVS 的 隧道（Tunnel）和网络地址转换（NAT）模式都属于网络层负载均衡，它们的主要区别在于对 IP 数据包的修改方式。

隧道模式中，LVS 会创建一个新的 IP 数据包，并将原始 IP 数据包整体放入新数据包的负载部分（Payload）中。随后，这个新数据包通过三层交换机发送出去。当后端服务器收到数据包时，通过相应的拆包机制去除负载均衡器添加的额外头部，解析出原始 IP 数据包。

举一个具体例子，假设客户端（IP 203.0.113.5）向 VIP (1.1.1.1) 发送的数据包如下：
```go
{
  Source IP: 203.0.113.5,
  Destination IP: 1.1.1.1,
  Payload: "Request data"
}
```

负载均衡器接收到数据包后，根据调度算法选择出一台后端服务器（172.12.1.3），然后对数据包执行封装处理。

```go
{
  Source IP: 172.12.1.2,
  Destination IP: 172.12.1.3,
  Payload: {
    Original Source IP: 203.0.113.5,
    Original Destination IP: 1.1.1.1,
    Original Data: "Request data"
  }
}
```

上述将一个 IP 数据包封装在另一个 IP 数据包内，并配合相应的解包机制，属于典型的 IP 隧道技术。Linux 中的 IPIP 隧道就是字面意思上的“IP in IP”。由于隧道模式工作在网络层，解除了直接路由模式的网络限制，因此 LVS 隧道模式可以跨越子网。

由于源数据包的所有信息均未被修改，隧道模式依然保留了三角传输模式的特征（当然，仍需处理 lo 回环地址）。隧道模式的请求到响应的整个过程如图 4-10 所示。

:::center
  ![](../assets/balancer4-tunnel.svg)<br/>
图 4-10 隧道模式的工作原理
:::

隧道（Tunnel）模式相当于直接路由模式的升级（支持了跨网）：
- 由于需要对数据包进行封装/解封，后端服务器必须支持某些隧道技术（如 IPIP、GRE）；
- 其次，**只要是三角模式（LVS 的直接路由模式或者隧道模式）必须保证后端服务器的的 lo 接口与负载均衡服务器有相同的虚拟 IP 地址**。因为回复客户端的数据包，必须使用 VIP 作为数据包的源地址，这样客户端收到之后才能正常处理。

## 4.4.3 网络地址转换模式

另一种对 IP 数据包的修改方式是**直接修改原始 IP 数据包的目标地址，将其替换为后端服务器的地址**。

这段解释可能不够直观，但相信大多数读者都曾操作过类似的配置。如在家中设置路由器时，你希望外部设备可以访问家中某台运行服务器的电脑。假设家中的电脑 IP 是 192.168.1.100，并在端口 8080 上运行一个 Web 服务。你可以在路由器中设置端口转发（NAT），将外部访问路由器的 80 端口的请求转发到该电脑的 192.168.1.100:8080。这样，当外部设备通过路由器的公共 IP 访问 80 端口时，实际上就会连接到局域网内的服务器。

**四层负载均衡器对 IP 数据包的改写与路由器中的“端口转发”原理相同**。因此，这种负载均衡方式被称为网络地址转换（NAT）模式，其请求和响应的流程如图 4-11 所示。

:::center
  ![](../assets/balancer4-NAT.svg)<br/>
图 4-11 网络地址转换（NAT）模式
:::

举例一个具体的例子，假设客户端（203.0.113.5:37118）请求负载均衡器（1.1.1.1:80），四层负载均衡器根据调度算法挑选了某个后端服务器（10.0.0.2:8080）处理请求。

此时，四层负载均衡器处理请求和响应的逻辑如下：
- 当客户端请求到达负载均衡器时，负载均衡器执行 NAT 操作：
	- 首先是 DNAT（目标地址转换） 操作：将目标 IP 和端口（1.1.1.1:80）改为后端服务器的 IP 和端口（10.0.0.2:8080），这使得**请求能够被路由至指定的后端服务器处理**。
	- 为了保持通信的完整性，负载均衡器还会执行 SNAT（源地址转换）操作。也就是原始源 IP 和端口（203.0.113.5:37118）改为四层负载均衡器的 IP 和端口（1.1.1.1:某个随机端口）。**SNAT 操作确保后端服务器认为请求是来自负载均衡器**，而不是直接来自客户端。
- 当后端服务器返回响应时，负载均衡器执行相反的 NAT 操作:
	- 将源 IP 和端口改回 1.1.1.1:80
	- 将目标 IP 和端口改回客户端的 203.0.113.5:37118

最终，客户端请求/接收的都是负载均衡器的 IP 和端口，并不知道实际的后端服务器信息。

从上述可见，网络地址转换模式下，负载均衡器代表整个服务集群接收和响应请求。因此，当流量压力较大时，系统的瓶颈就很容易体现在负载均衡器上。

## 4.4.4 主备模式

到目前为止，我们讨论的都是单个负载均衡器的工作模式。假如负载均衡器挂了呢？那所有经过负载均衡器的连接都会受到影响。为了避免负载均衡器故障导致服务停摆，负载均衡器通常都是以高可用模式部署。


图 4-12 展示的是最常见主备模式。其核心在于每台节点中运行着 Keepalived 软件。该软件实现了 VRRP（Virtual Router Redundancy Protocol）协议，虚拟出一个 IP 地址（VIP）作为对外服务的地址。该 VIP 绑定默认绑定在主节点上，由主节点（Master）处理所有流量请求。备用节点（Backup）始终监测主节点状态，并在故障时迅速接管 VIP，以保证服务不中断。

:::center
  ![](../assets/lvs-ha.svg)<br/>
  图 4-12 主备模式
:::

主/备模式的设计在现今的分布式系统非常普遍，但这种方式存在下述缺陷：
- 平正常运行时，50% 的资源处于闲置状态，备用服务器一直空转，导致资源利用率不足。
- 此外，现代分布式系统更注重高容错性。理想情况下，即便多个实例同时故障，服务仍应能持续运行。而在主备模式下，一旦主备实例同时故障，服务将完全中断。


## 4.4.5 基于集群和一致性哈希的容错和可扩展模式

近些年，大型互联网基础设施系统开始设计和部署全新的大规模并行四层负载均衡系统。这些系统的设计目标是：
- 避免主备模式的缺点。
- 从厂商的商业硬件方案，迁移至基于标准服务器和网卡的通用软件方案。

如图 4-13 所示，这种设计被称“基于集群和一致性哈希的容错和可扩展”（fault tolerance and scaling via clustering and distributed consistent hashing）。它的工作原理如下：

- N 个边缘路由器使用相同的 BGP 权重通告所有 Anycast VIP。通过 ECMP（Equal-cost Multi-path Routing）确保同一流（flow）的所有数据包都经过相同的边缘路由器。一个流通常由 4 元组（源 IP/端口和目的 IP/端口）组成。简而言之，ECMP 是一种通过一致性哈希将流量均匀分发到多台权重相同的路由器的机制。虽然边缘路由器通常不关心每个数据包的具体去向，但它们希望同一流的所有包始终走相同路径，以避免因乱序而导致性能下降。

- N 个四层负载均衡器使用相同的 BGP 权重向所有边缘路由器通告所有 VIP。通过 ECMP，边缘路由器确保相同流（flow）的数据包始终选择相同的四层负载均衡器。
- 每个四层负载均衡器实例，使用一致性哈希算法为每个流（flow）选择一个后端。

:::center
  ![](../assets/balancer-ha-2.svg)<br/>
  图 4-13 基于集群和一致性哈希的容错和可扩展模式
:::

综合上述，看看一致性哈希容错模式是如何避免主备方式的缺陷：
- 边缘路由器和负载均衡器实例可以根据需求动态扩展。每一层都使用 ECMP，确保新实例加入时，受影响的流量（flow）数量最小化。
- 在预留足够的突发量和容错空间的基础上，系统的资源利用率可以根据需求达到最优水平。
- 无论是边缘路由器，还是负载均衡器均可基于通用硬件构建，其成本仅为传统硬件负载均衡器的一小部分。

绝大部分的现代四层负载均衡系统都在朝着这种设计演进，像各类云厂商的负载均衡系统，绝大部分都是这种设计。 