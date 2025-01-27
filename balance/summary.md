# 第四章：负载均衡技术

:::tip <a/>
当你试图解释一条命令、一个语言特性或是一种硬件的时候，请首先说明它要解决的问题。

:::right 
—— 摘自《编程珠玑》[^1]
:::

无论是出于扩展服务能力的考虑，还是提高容错能力的考虑，大多数系统通常以集群形式对外提供服务。

以集群形式对外提供服务时，用户请求无论由哪台服务器处理，都应获得一致的结果。另一方面，集群还需对用户保持足够的度透明。也就是说，用户与集群交互时仿佛面对一台高性能、高可用的单一服务器。集群内部动态增加或删除服务器时，用户不会察觉这些变化，也无需修改任何配置。

为集群提供统一入口并实现上述职责的组件称为负载均衡（或称代理）。负载均衡是业内最活跃的领域之一，涉及的产品层出不穷（如专用网络设备、基于通用服务器的软件等），部署拓扑多样（如中间代理型、边缘代理、客户端内嵌等）。无论其形式如何，所有负载均衡的核心职责无外乎 “选择处理用户请求的目标”（即负载均衡算法）和“将用户请求转发至目标”（即负载均衡的工作模式）。本章我们围绕这两个核心职责，分析负载均衡的原理与工作模式，掌握各类负载均衡应用技术。
:::center
  ![](../assets/balance-summary.png)<br/>
  图 4-0 本章内容导读
:::

[^1]:《编程珠玑》作者是 Jon Bentley，被誉为影响算法发展的十位大师之一。在卡内基-梅隆大学担任教授期间，他培养了包括 Tcl 语言设计者 John Ousterhout、Java 语言设计者 James Gosling、《算法导论》作者之一Charles Leiserson 在内的许多计算机科学大家。