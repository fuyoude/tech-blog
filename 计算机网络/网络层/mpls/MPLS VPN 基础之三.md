# MPLS VPN 基础

## 7.MPLS L3VPN 中的 OSPF VPN 扩展机制

### 7.1 OSPF 超级骨干区域

OSPF 协议是 PE 和 CE 之间路由协议的另一种选择，如果要将 VPN 用户路由传递到其他 PE，**<font color="red">那么需要在 PE 上把 OSPF 路由引入到 MP-BGP，在远端 PE 上需要将 MP-BGP 的 VPNv4 路由引入到 OSPF</font>**。在远端站点的 PE 上将 MP-BGP 的 VPNv4 路由引入到 OSPF 后，这些路由变成了 OSPF 外部路由（从 BGP 引入到 OSPF），导致路由的优先级变低，这样会带来以下两个问题。

- 如果此时客户站点之间拥有备份链路（后门链路），如下图所示，当两个客户站点的 PE-CE 链路及站点间的后门链路均属于同一个 OSPF 区域时，**<font color="red">CE 可以通过后门链路学习到远端站点的 OSPF 区域内路由（O）</font>**。与此同时，**<font color="red">远端站点的 OSPF 路由通过 MP-BGP VPNv4 在 PE 之间传播后，为 OSPF 外部路由（ASE）</font>**。这会导致远端站点优先选择后门链路的路由，而不是穿越 MPLS VPN 骨干的路由。这不符合用户起初的期望。

<div align="center">
    <img src="mpls_static/53.png" width="400"/>
</div>

- 另外，从网络设计角度考虑，这种情况下使得同个 VPN 用户的不同站点被 MPLS VPN 骨干分割了。

为避免在远端 PE 上将从 BGP 引入 OSPF 中的路由变成外部路由，MPLS VPN 在远端 PE 上将 MP-BGP 传递过来的 VPNv4 路由引入到 OSPF 时，以 LSA3 的形式通告出来（常规 OSPF 的做法是以 LSA5 的形式通告出来，PE 是 ASBR），如下图所示。这样的设计就让 MPLS VPN 骨干网看起来像一个 OSPF 的骨干区域（PE 成了 ABR）。对于 OSPF 来说，我们称之为超级骨干区域。从网络设计角度看，这就由中间的一个超级骨干区域将多个站点的 OSPF 区域连接在了一起，从而形成了一个整体。·1

<div align="center">
    <img src="mpls_static/54.png" width="550"/>
</div>

总的来说，在传统的 OSPF 网络中，不同区域通常通过骨干区域 Area 0 实现路由交换，ABR 负责连接不同区域，并通过 Type 3 LSA 通告区域间路由。例如，一条路由从 Area 1 经过 Area 0 传递到 Area 2，Area 2 中的路由器通常会将其识别为 OSPF 区域间路由，而不是外部路由。

但是，当企业的多个站点通过 MPLS L3VPN 互联时，各站点之间不再通过传统 OSPF 骨干直接交换路由，而是需要由本地 PE 从 CE 学习 OSPF 路由，再转换成 VPNv4 路由，通过 MP-BGP 传递到远端 PE，最后由远端 PE 重新发布到客户 OSPF 网络。如果按照普通的 BGP 到 OSPF 路由重分发机制处理，远端 PE 通常会将从 MP-BGP 学到的路由作为 OSPF 外部路由。这样，原本属于同一个企业 OSPF 域的内部路由，在经过 MPLS VPN 后就变成了外部路由，可能改变原有的 OSPF 选路结果。

为了解决这个问题，RFC 4577 将 MPLS VPN 骨干在逻辑上视为客户 OSPF 网络的一个**超级骨干区域（Superbackbone）**。

对于 OSPF 来说，有了这个超级骨干区域后，用户各站点内的 OSPF 区域设计需要注意以下几点。

- **<font color="red">用户站点内如果是单区域，各站点可以使用相同或不同的区域</font>**，如下图所示。

<div align="center">
    <img src="mpls_static/55.png" width="270"/>
    <img src="mpls_static/56.png" width="250"/>
</div>

- **<font color="red">用户站点内部如果采用多区域，要确保骨干区域（Area0）连接在 PE 上</font>**，如下图所示。

<div align="center">
    <img src="mpls_static/57.png" width="350"/>
</div>

如果用户站点内部采用多区域，而骨干区域（Area0）没有连接在 PE 上，那么会发生什么呢？（如下图所示）。

<div align="center">
    <img src="mpls_static/58.png" width="350"/>
</div>

**这样会导致用户站点 Area0 学不到其他站点的路由**，上述网络拓扑图的逻辑结构如下：

```java{.line-numbers}
                      MPLS VPN Backbone
                   （MP-BGP 超级骨干区域）
                       │           │
                      PE1         PE2
                       │           │
                     Area 1      Area 2
                       │           │
                      CE1         CE2
                       │           │
                     Area 0      Area 0
                       │           │
                     Site1       Site2
```

**（1）Site1 发布路由到 Site2**

假设 Site1 的 Area 0 中存在一条网络 **`10.10.10.0/24`**，现在希望将这条路由通过 MPLS VPN 传递给 Site2。由于 CE1 同时连接 Area 0 和 Area 1，它是一个普通的 OSPF ABR，CE1 首先通过 Area 0 内部的 **`Type 1/Type 2`** 等 LSA 学习到 **`10.10.10.0/24`**。随后，CE1 向 Area 1 生成一条 Type 3 LSA，PE1 可以接收到这条 Type3 LSA。

PE1 收到了 Type3 LSA，但是 PE1 作为超级骨干区域中的 ABR。**<font color="red">因为 ABR 在非骨干区域收到 Type-3 LSA，虽然它会将其装载进 LSDB，但是该路由器不会使用这些 Type-3 LSA 进行路由计算</font>**（不会进全局路由表），当然它更不会将这些 Type-3 LSA 再注入回 Area0 中。并且 ABR 从非骨干区域发往骨干区时，只能通告 **`intra-area routes`**。

因此 PE1 可以在 Area 1 的 LSDB 中看到 **`10.10.10.0/24`** 对应 Type 3 LSA，但它不会根据这条来自非骨干区域的 Type 3 LSA 计算出相应的区域间路由。因此，由于 PE1 没有学到有效路由，也就无法通过 MP-BGP 正常发布，PE2 和 Site2 也就无法正常学习到它。

**（2）Site2 发布路由到 Site1**

假设 CE2 在 Area 2 中存在网络 **`192.168.20.0/24`**，PE2 可以通过 Area 2 的 OSPF 正常学习这条区域内路由，并将其引入 MP-BGP，再通过 VPNv4 路由传递给 PE1。

PE1 收到该路由后，如果 OSPF Domain ID 匹配、原始路由类型正确，就可以向本地 Area 1 生成 Type 3 LSA。此时，Area 1 中的普通内部路由器可以根据 Type3 LSA 学到 **`192.168.20.0/24`**。但是 CE1 同时连接 Area 0 和 Area 1，是一个 ABR。因此 CE1 不会利用从非骨干区域 Area 1 收到的 Type 3 LSA 计算区域间路由。

所以，即使 PE1 已经把远端 Site2 的路由作为 Type3 LSA 发布到了 Area 1，CE1 仍然不会将它通告到自己连接的 Area 0 中。

### 7.2 用于 OSPF 的 BGP 扩展团体属性

要想 OSPF 路由能无缝地穿越 MPLS VPN 骨干网络，就必须使 MP-BGP 协议携带 OSPF 路由的特征，比如路由类型、进程号等，**<font color="red">这些特征可以帮助远端 PE 决定向 CE 端通告什么类型的 LSA</font>**，这样就使得 OSPF 路由可以在远端 PE 处得以完全重建。

MPLS VPN 体系结构中使用 BGP 的扩展团体属性来传递 OSPF 路由的特征，这些特征包括：

- 路由类型；
- 区域号；
- OSPF 路由器 ID；
- 域 ID；
- OSPF 外部路由的度量值类型 1 或 2。

在 MPLS L3VPN 的 OSPF VPN 扩展机制中，PE 将 VPNv4 路由重新发布到 OSPF 时，需要根据路由携带的 OSPF Domain ID 和 OSPF Route Type 扩展团体属性，共同判断该路由应当以哪种 LSA 类型进行通告。

**<font color="red">如果路由的 Domain ID 与本地 OSPF 进程的 Domain ID 匹配，并且 Route Type 为 1、2、3，则远端 PE 通常以 Type 3 LSA 的形式将该路由通告给 CE；如果 Domain ID 不匹配，或者原始路由属于 OSPF 外部路由，则将其按照外部路由处理，通常以 Type 5 LSA 的形式发布</font>**。

OSPF 进程的域 ID 包含在此进程生成的路由中，在将 OSPF 路由引入到 BGP 中时，域 ID 被附加到 VPNv4 路由上，作为 BGP 的扩展团体属性传递。根据 RFC 2328，A 32-bit number identifying the area. The Area ID of **`0.0.0.0`** is reserved for the backbone. 根据 RFC 4577，each OSPF instance MUST be associated with one or more Domain Identifiers. Area numbers are relative to a particular OSPF domain. 也就是说 OSPF 区域编号是相对于某个特定的 OSPF 路由域而言的。

<div align="center">
    <img src="mpls_static/59.png" width="550"/>
</div>

在之前 PE-CE 之间运行 OSPF 协议的网络拓扑图，PE2 上 VPNv4 路由（**`1.1.5.1/32`**）的路由更新，以及为该路由所关联的 BGP 扩展属性如下所示。

```java{.line-numbers}
<AR4>display bgp vpnv4 all routing-table 1.1.5.1 32

 BGP local router ID : 10.4.4.4
 Local AS number : 100

 Total routes of Route Distinguisher(100:1): 1
 BGP routing table entry information of 1.1.5.1/32:
 Label information (Received/Applied): 1028/NULL
 From: 10.1.1.1 (10.1.1.1)
 Route Duration: 00h00m19s  
 Relay IP Nexthop: 10.1.34.3
 Relay IP Out-Interface: GigabitEthernet0/0/0
 Relay Tunnel Out-Interface: GigabitEthernet0/0/0
 Relay token: 0x1
 Original nexthop: 10.1.1.1
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <10.1.1.1 : 0>
 AS-path Nil, origin incomplete, MED 2, localpref 100, pref-val 0, valid, internal, best, select, pre 255, IGP cost 3
 Not advertised to any peer yet
```

关于上面的输出内容，BGP 扩展团体属性的解释如下。

- **`RT<1:1>`**：这个值表示 VPN 路由的 RT 值，注意这里不是表示路由类型（route-type），而是 vpn-target，也就是 **`export RT`** 值为 **`1:1`**。
- **`OSPF DOMAIN ID<0.0.0.0:0>`**：冒号前面部分 **`0.0.0.0`** 表示域 ID 值，后面部分表示可选项，默认为 0。
- **`OSPF RT<0.0.0.0:1:0>`**：这个值表示 OSPF 路由类型，共分成 3 部分，第 1 部分表示区域号，第 2 部分表示路由类型，第 3 部分表示外部路由的度量值类型 1 或 2。后两个部分（路由及度量值类型）取值及其解释如下。
  - **`:1:0`** or **`:2:0`** 表示内部路由（LSA1 或 LSA2）。
  - **`:3:0`** 表示区域间路由（LSA3）。
  - **`:5:0`** or **`5:1`** 表示外部路由（LSA5），度量值类型分别是 1 或 2。
  - **`:7:0`** or **`7:1`** 表示外部路由（LSA7），度量值类型分别是 1 或 2。
  - **`:129:0`** 表示 sham-link 端点。
- **`OSPF Router-ID<10.1.1.1:0>`** 表示发送该路由的 VPN 实例的 Router-ID。

### 7.3 OSPF 路由穿越 MPLS VPN 骨干网的 metric 传递

在 PE 上将 OSPF 内部路由或外部路由引入到 BGP 后，使用 OSPF 路由的 Cost 值来设置 BGP 路由的 MED 值。当远端的 PE 把 BGP 路由引入到 OSPF 区域时，又将 MED 值复制回来给 OSPF 路由的 Cost 值，具体过程分为以下几种情况。

（1）两端 PE 的域 ID 设置相同的情况，OSPF 内部和外部路由的 Cost 传递的情况如下图所示。

<div align="center">
    <img src="mpls_static/60.png" width="600"/>
</div>

如上图所示，PE1 收到 CE1 的路由后，**因为要累加入口链路的开销，所以引入到 BGP 后的 MED 值为 2**。远端的 PE2 将 BGP 路由引入到 OSPF 区域时，直接用 MED 值代替 OSPF 路由的开销值。这种情况下，由于 PE1 和 PE2 的域 ID 相同，所以在 PE2 上会根据路由的类型来产生相应的 LSA 给 CE2。

（2）两端 PE 的域 ID 设置不相同的情况，OSPF 内部和外部路由的 Cost 传递的情况如下图所示。

<div align="center">
    <img src="mpls_static/61.png" width="600"/>
</div>

如上图所示，因为 PE1 和 PE2 的域 ID 不同，所以在 PE2 上把 BGP 路由引入到 OSPF 区域的路由都是 OSPF 外部路由，而 Cost 值仍然是复制 BGP 路由的 MED 值。

>华为设备的 **`DOMAIN-ID`** 值默认为 0，它和 VPN 实例中的 OSPF 进程 ID 大小没有关系。

### 7.4 防止 OSPF 站点间的路由环路

在对网络健壮性要求较高的应用中，可以采用 CE 双归属方式组网，如下图所示。

<div align="center">
    <img src="mpls_static/62.png" width="450"/>
</div>

但是在 CE 双归属场景中，用户的 VPN 路由有可能通过多个 PE 传播到公网，导致潜在路由环路的产生，下面是产生环路的案例。

<div align="center">
    <img src="mpls_static/63.png" width="550"/>
</div>

如上图所示，PE1、PE2、PE3 都已建立 iBGP 连接，PE 和 CE 之间使用 OSPF 路由协议。这时假设由于链路质量问题，PE1 先接收到来自 PE3 的 VPNv4 路由（**`2.2.2.2/32`**），PE1 会将 BGP 路由引入到左侧站点的 OSPF 区域，PE2 从 OSPF 学习到该路由，下一跳指向 CE1。然后又在 PE 上做了双向引入，所以 PE2 会将该 OSPF 路由引入到 BGP 形成 VPNv4 路由，同时将该路由通告给 PE1。这时，PE1 从 PE2 和 PE3 接收到了同一条 VPN 路由，在链路开销相等的情况下，假设 PE2 的 **`Router-ID`** 小于 PE3 的 **`Router-ID`**，那么 PE1 会优选 PE2 为下一跳，环路就产生了。

为避免潜在的路由环路问题，需要使用到 OSPF 报文头部选项中的 DN 比特位。DN 比特位于 **`LSA3/LSA5/LSA7`** 中的 Option 字段，用于表明路由是由 PE 向 CE 方向传递的，**<font color="red">也就是说一条路由如果由 PE 发布给 CE，这条路由的 DN 比特位就会被置位为 1</font>**。同一用户站点的其他 PE 在收到带有置了 DN 比特位的 LSA3 时，不会进行 SPF 路由计算，更不会将其引入到 BGP 中。如下图所示，显示了一台 PE 路由器在接收到置了 DN 比特位的 LSA 时，不会再将其通告进 BGP 中。

<div align="center">
    <img src="mpls_static/64.png" width="400"/>
</div>

### 7.5 普通 CE 会接收 DN 置位的 LSA 并计算路由

<div align="center">
    <img src="mpls_static/45.png" width="750"/>
</div>

在上述网络拓扑中，PE-CE 之间运行 OSPF 协议，远端站点 CE2 的客户路由 **`1.1.6.1/32`** 可能先通过 OSPF 交给 PE2，再由 PE2 转换成 MP-BGP VPN 路由穿越 VPN Backbone。当这条 VPN 路由到达 PE1 后，PE1 又把它转换成 OSPF 路由发送给 CE1。当 PE1 把这种 VPN 路由重新发布成 **`Type 3/Type 5`** 传递给 CE1 时，会在相应 LSA 中设置 DN Bit。通过下面的输出可以看到，PE1 发布给 CE1 的 **`1.1.6.1/32`** Type3 LSA 带有 DN 标志位。

```java{.line-numbers}
<AR1>display ospf lsdb summary 1.1.6.1
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Sum-Net
  Ls id     : 1.1.6.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 107 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 80000003 
  chksum    : 0x1fa3
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Low
```

然后 CE1 作为普通的 CE 会接收这个 Type3 LSA，然后计算并形成 OSPF 路由，如下所示。

```java{.line-numbers}
<AR5>display ospf lsdb summary 1.1.6.1
	 OSPF Process 1 with Router ID 5.5.5.5
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Sum-Net
  Ls id     : 1.1.6.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 181 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 80000003 
  chksum    : 0x1fa3
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Medium
<AR5>display ospf routing 
	 OSPF Process 1 with Router ID 5.5.5.5
		  Routing Tables 

 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 1.1.5.1/32         0     Stub       1.1.5.1         5.5.5.5         0.0.0.0
 172.16.1.0/24      1     Transit    172.16.1.2      5.5.5.5         0.0.0.0
 1.1.6.1/32         3     Inter-area 172.16.1.1      10.1.1.1        0.0.0.0

 Routing for ASEs
 Destination        Cost      Type       Tag         NextHop         AdvRouter
 172.16.2.0/24      1         Type2      3489661028  172.16.1.1      10.1.1.1

 Total Nets: 4  
 Intra Area: 2  Inter Area: 1  ASE: 1  NSSA: 0 
```

根据华为的官方文档，DN-bit It is a flag bit used by OSPF multi-instance processes to prevent routing loops. **<font color="red">_After OSPF multi-instanceis configured on the router (a PE or an MCE)_</font>**, the router sets the DN-bit of generated Type 3, Type 5, or Type 7 LSAs to 1 and retains the DN-bit (0) of other LSAs. When calculating routes, the OSPF multi-instance process on the router ignores LSAs with DN-bit 1, which prevents the router from receiving the LSAs that are advertised by itself. **<font color="red">也就是针对华为的设备，执行 DN Bit 检查的是 OSPF 多实例进程，典型设备是 PE 或 MCE</font>**。

### 7.6 MCE 未关闭 DN Bit 检查:LSA 入库但不生成路由

接下来我们对 CE1 的配置进行更改，使其从普通的 CE 变为 MCE。具体的 MCE 配置如下所示：

```java{.line-numbers}
#
 sysname AR5
#
ip vpn-instance vpna
 ipv4-family
  route-distinguisher 100:5
#
interface GigabitEthernet0/0/0
 ip binding vpn-instance vpna
 ip address 172.16.1.2 255.255.255.0 
#
interface LoopBack0
 ip binding vpn-instance vpna
 ip address 1.1.5.1 255.255.255.255 
#
ospf 1 router-id 5.5.5.5 vpn-instance vpna
 area 0.0.0.0 
  network 1.1.5.1 0.0.0.0 
  network 172.16.1.0 0.0.0.255 
```

上述就是 MCE 的配置，首先是创建 VPN 实例 vpna，并且和 OSPF 进程 1 绑定。根据华为的官方文档，An OSPF process can be bound to only one VPN instance. If an OSPF process is not bound to any VPN instance before it is started, this process becomes a public network process and cannot be bound to a VPN instance later. **<font color="red">一个 OSPF 进程只能绑定一个 VPN 实例（也就是不能让同一个 OSPF 进程同时属于 vpna 和 vpnb）</font>**，而同一台路由器可以运行多个独立的 OSPF 进程。

接下来，在 VPN 实例 vpna 中，配置了 **`route-distinguisher 100:5`**，这是因为只有配置了 **`route-distinguisher`** 之后，VPN 实例的 IPv4 地址族才能生效。根据华为的文档，A VPN instance IPv4 address family takes effect only after being assigned an RD. The RDs of different VPN instance IPv4 address families on a PE must be different.

但是在 VPN 实例 vpna 中，没有配置 vpn-target，这是因为 RT（Route Target）是 BGP 扩展团体属性（Extended Community）的一种，广泛用于 BGP/MPLS L3VPN 的 VPN 路由导入、导出控制。根据华为的文档，skip this step when the device functions as the MCE only. 也就是说如果设备仅作为 MCE 使用，**`vpn-target`** 这一步可以跳过。

上述配置以后，其余 AR 路由器的配置不发生变化，并且 CE1 也不配置 **`vpn-instance-capability simple`**，此时 CE1 上的 OSPF 路由表如下所示，可以看到没有 **`1.1.6.1/32`** 路由，但是收到了 **`1.1.6.1/32`** 的 Type3 LSA。
```java{.line-numbers}
<AR5>display ospf routing 

	 OSPF Process 1 with Router ID 5.5.5.5
		  Routing Tables 

 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 1.1.5.1/32         0     Stub       1.1.5.1         5.5.5.5         0.0.0.0
 172.16.1.0/24      1     Transit    172.16.1.2      5.5.5.5         0.0.0.0

 Total Nets: 2  
 Intra Area: 2  Inter Area: 0  ASE: 0  NSSA: 0 
<AR5>display ospf lsdb summary 1.1.6.1

	 OSPF Process 1 with Router ID 5.5.5.5
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Sum-Net
  Ls id     : 1.1.6.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 393 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 8000000a 
  chksum    : 0x11aa
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Medium
```

MCE 本质上仍属于客户侧设备，但为了同时承载并隔离多个 VPN，需要为不同 VPN 配置独立的 VPN 实例，并运行与之绑定的 OSPF 进程。**<font color="red">因此，MCE 既需要像普通 CE 一样学习 PE 发布的远端 VPN 路由，又具备 OSPF VPN 多实例的防环检测机制</font>**。

### 7.7 MCE 关闭 DN Bit 检查:LSA 入库并且生成路由

在 CE1 上进行如下配置：

```java{.line-numbers}
ospf 1 router-id 5.5.5.5 vpn-instance vpna
 vpn-instance-capability simple
```

**`vpn-instance-capability simple`** 用于禁止路由环路检测，直接进行路由计算。在配置好该命令之后，CE1 上收到了 **`1.1.6.1/32`** 的 Type3 LSA（DN bit 置位）。但是在 OSPF 路由表中存在 **`1.1.6.1/32`** 的路由，也就是说，配置这个命令以后，OSPF 不再检查 DN Bit 和 Route Tag，而直接计算 OSPF 路由。

```java{.line-numbers}
[AR5-ospf-1]display ospf routing 
	 OSPF Process 1 with Router ID 5.5.5.5
		  Routing Tables 
 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 1.1.5.1/32         0     Stub       1.1.5.1         5.5.5.5         0.0.0.0
 172.16.1.0/24      1     Transit    172.16.1.2      5.5.5.5         0.0.0.0
 1.1.6.1/32         3     Inter-area 172.16.1.1      10.1.1.1        0.0.0.0
 Routing for ASEs
 Destination        Cost      Type       Tag         NextHop         AdvRouter
 172.16.2.0/24      1         Type2      3489661028  172.16.1.1      10.1.1.1

 Total Nets: 4  
 Intra Area: 2  Inter Area: 1  ASE: 1  NSSA: 0 
[AR5-ospf-1]display ospf lsdb summary 1.1.6.1

	 OSPF Process 1 with Router ID 5.5.5.5
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Sum-Net
  Ls id     : 1.1.6.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 237 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 8000000c 
  chksum    : 0xdac
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Medium
```

在 MPLS L3VPN 中，PE 将从 VPN 骨干学习的 BGP 路由重新发布到 OSPF 时，会为生成的 Type 3、Type 5 或 Type 7 LSA 设置 **`DN=1`**，以防止路由环路。需要注意，DN Bit 并不阻止 LSA 的接收、存储和正常泛洪，而是影响 OSPF VPN 多实例进程的路由计算。

在 **`PE1—MCE—PE2`** 的组网中，MCE 可以接收 PE1 发布的 **`DN=1`** LSA，并将其保存到 LSDB。如果 MCE 未配置 **`vpn-instance-capability simple`**，其 OSPF VPN 多实例进程默认会进行 DN Bit 防环检查，不使用此类 LSA 计算路由，导致 LSDB 中存在 LSA，但路由表中没有对应路由。

在 MCE 对应的 OSPF 进程下配置 **`vpn-instance-capability simple`** 后，可以关闭 DN Bit 和 VPN Route Tag 相关的路由防环检查，使携带 **`DN=1`** 的 LSA 能够正常参与计算并生成路由。该命令使 MCE 在保留多 VPN 隔离能力的同时，具备普通 CE 所需的路由学习行为。

### 7.8 跨区域重新生成 LSA3 导致 DN Bit 丢失

DN 位也无法避免上述的环路问题，如下图所示的场景，CE 双归属，CE1 连接 PE1 的链路在 Area0 中，CE1 连接 PE2 的链路放在 Area1 中。当 PE1 以 LSA3 的形式向 CE1 发送 OSPF 路由时，这时 DN 比特置位。由于 CE1 是 ABR，当这条 LSA3 经过 CE1 再传递到 Area1 时，DN 比特位置为 0，当 PE2 接收后，可能仍然会计算该路由，从而造成环路隐患。

<div align="center">
    <img src="mpls_static/65.png" width="600"/>
</div>

#### 7.8.1 实验配置

上述网络拓扑图中，AR1 的配置如下所示，

```java{.line-numbers}
#
 sysname AR1
#
ip vpn-instance vpna
 ipv4-family
  route-distinguisher 100:1
  vpn-target 1:1 export-extcommunity
  vpn-target 1:1 import-extcommunity
#
mpls lsr-id 10.1.1.1
mpls
#
mpls ldp
#
interface GigabitEthernet0/0/0
 ip binding vpn-instance vpna
 ip address 10.1.14.1 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 10.1.12.1 255.255.255.0 
 mpls
 mpls ldp
#
interface LoopBack0
 ip address 10.1.1.1 255.255.255.255 
#
bgp 100
 router-id 10.1.1.1
 undo default ipv4-unicast
 peer 10.3.3.3 as-number 100 
 peer 10.3.3.3 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  undo peer 10.3.3.3 enable
 # 
 ipv4-family vpnv4
  policy vpn-target
  peer 10.3.3.3 enable
 #
 ipv4-family vpn-instance vpna 
  import-route ospf 2
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 10.1.1.1 0.0.0.0 
  network 10.1.12.0 0.0.0.255 
#
ospf 2 router-id 10.1.1.1 vpn-instance vpna
 import-route bgp
 area 0.0.0.0 
  network 10.1.14.0 0.0.0.255 
```
**（1）配置 `ospf 2 router-id 10.1.1.1 vpn-instance vpna`**

首先解释一下配置 **`ospf 2 router-id 10.1.1.1 vpn-instance vpna network 10.1.14.0 0.0.0.255`**。network 只能匹配已经属于 vpna 的接口，**`network 10.1.14.0 0.0.0.255`** 的前提是 AR1 的 **`GE0/0/0`** 接口必须先配置 **`ip binding vpn-instance vpna`**。绑定之后，ospf 2 只会去扫描属于 vpna 的接口，找到 **`10.1.14.1`** 并激活 OSPF，从而与 AR4（CE1）建立邻居。如果 **`GE0/0/0`** 没绑定 vpna（留在公网里），就算 ospf 2 写了这条 network，也不会生效。

另外，AR1 通过 ospf 2 从 AR4 学到的所有私网路由（例如 AR4 的环回口 **`10.4.4.4/32`**），只会写入 vpna 的路由表，绝对不会出现在 AR1 的公网全局路由表里。这样即使有另一个客户 vpnb 也用了 **`10.4.4.4/32`**，两者也不会冲突。

**（2）import-route bgp**

首先 AR3 学到 AR5 的私网路由后，会给它加上 RD、私网 MPLS 标签，以及 AR3 上 vpna 配置的 Export RT，封装成一条 VPNv4 路由，通过 MP-BGP 传给 AR1。AR1 收到这条 VPNv4 路由后，会提取它携带的 Export RT，与本地 vpna 实例中配置的 Import RT 进行比对。如果不匹配就直接丢弃，不进入 vpna。如果匹配，AR1 就剥离掉 RD，将其还原为普通的 IPv4 路由。

还原后的路由会先进入 BGP 的 **`ipv4-family vpn-instance vpna`** 路由表。如果它经过 BGP 选路后成为最优路由，就会被下发到 vpna 的 IP 路由表中，协议类型显示为 IBGP。由于 ospf 2 绑定了 vpna，**<font color="red">它里面的 **`import-route bgp`** 命令只会去扫描 vpna 路由表中处于活跃（Active）状态的 BGP 路由，把它们转换成 OSPF 2 的 LSA</font>**，再通过运行 OSPF 2 协议接口 **`G0/0/0`** 发给 AR4。

**（3）import-route ospf**

因为 ospf 2 在创建时就已经绑定了 vpna 实例，所以 AR1 通过 ospf 2 从 AR4 学到路由（比如 **`10.4.4.4/32`**）后，直接就把它放进了 vpna 的 IP 路由表中，协议类型为 OSPF。当在 bgp 100 的 **`ipv4-family vpn-instance vpna`** 视图下敲入 **`import-route ospf 2`** 时，BGP 进程就会去扫描 vpna 的 IP 路由表，执行筛选。**<font color="red">首先只看 vpna 实例路由表（不会去查询 public 路由表），接下来只在 vpna 路由表中挑选协议类型为 OSPF 2，且状态为 Active（活跃） 的路由</font>**。最后将将这些挑选出来的路由安装进 BGP 的 vpna 路由表中（可以通过 **`display bgp vpnv4 vpn-instance vpna routing-table`** 进行查看）。

一旦路由进入了 BGP 的 vpna 表，BGP 就会自动加上私网标签、RD 以及 export RT 等，变成一条 VPNv4 路由发送给远端 MP-BGP 邻居。 

因此，总结为 import-route bgp 的作用就是把 vpna 路由表里活跃的 BGP 路由引入到 ospf 2 的 LSDB 中，再发给 AR4。import-route ospf 2 的作用就是把 vpna 路由表里活跃的 OSPF 2 路由引入到 BGP 的 vpna 路由表中，方便后续添加 RD、私网标签和 Export RT，发给对端 AR3。vpna IP 路由表作为 BGP VPN 和 OSPF 2 共同的路由空间和转换媒介，不过 OSPF 和 BGP 各自还有协议内部的路由数据库。

**（4）undo default ipv4-unicast**

因为 AR1 和 AR3（**`10.3.3.3`**）之间只需要传递 vpnv4 私网路由，不需要传递公网 IPv4 路由，所以用 **`undo default ipv4-unicast`** 关掉了默认的公网邻居建立，只和 AR3 建立 vpnv4 的连接，这样更干净安全。

AR3 的配置如下所示：

```java{.line-numbers}
#
 sysname AR3
#
ip vpn-instance vpna
 ipv4-family
  route-distinguisher 100:3
  vpn-target 1:1 export-extcommunity
  vpn-target 1:1 import-extcommunity
#
mpls lsr-id 10.3.3.3
mpls
#
mpls ldp
#
interface GigabitEthernet0/0/0
 ip binding vpn-instance vpna
 ip address 10.1.34.3 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 10.1.23.3 255.255.255.0 
 mpls
 mpls ldp
#
interface GigabitEthernet0/0/2
 ip binding vpn-instance vpna
 ip address 10.1.35.3 255.255.255.0 
#
interface LoopBack0
 ip address 10.3.3.3 255.255.255.255 
#
bgp 100
 router-id 10.3.3.3
 undo default ipv4-unicast
 peer 10.1.1.1 as-number 100 
 peer 10.1.1.1 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  undo peer 10.1.1.1 enable
 # 
 ipv4-family vpnv4
  policy vpn-target
  peer 10.1.1.1 enable
 #
 ipv4-family vpn-instance vpna 
  import-route ospf 2
#
ospf 1 router-id 3.3.3.3 
 area 0.0.0.0 
  network 10.1.23.0 0.0.0.255 
  network 10.3.3.3 0.0.0.0 
#
ospf 2 router-id 10.3.3.3 vpn-instance vpna
 import-route bgp
 area 0.0.0.0 
  network 10.1.35.0 0.0.0.255 
 area 0.0.0.1 
  network 10.1.34.0 0.0.0.255 
```

#### 7.8.2 实验结果

我们先在 AR3 上执行如下命令，确保 CE2 的路由 **`1.1.5.1/32`** 已经到达 AR3。接下来，AR3 将 vpna 中的 **`1.1.5.1/32`** 路由添加上 RD（**`100:3`**）、export RT（**`1:1`**）以及私有标签 1028 通过 MP-BGP 传递给 PE1。

```java{.line-numbers}
<AR3>display ospf 2 routing 
	 OSPF Process 2 with Router ID 10.3.3.3
		  Routing Tables 

 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 10.1.34.0/24       1     Transit    10.1.34.3       10.3.3.3        0.0.0.1
 10.1.35.0/24       1     Transit    10.1.35.3       10.3.3.3        0.0.0.0
 1.1.5.1/32         1     Stub       10.1.35.5       5.5.5.5         0.0.0.0
 10.5.5.5/32        1     Stub       10.1.35.5       5.5.5.5         0.0.0.0

 Total Nets: 4  
 Intra Area: 4  Inter Area: 0  ASE: 0  NSSA: 0 
<AR3>display ip routing-table vpn-instance vpna 1.1.5.1
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  OSPF    10   1           D   10.1.35.5       GigabitEthernet0/0/2
<AR3>display bgp vpnv4 all routing-table 1.1.5.1

 BGP local router ID : 10.3.3.3
 Local AS number : 100

 Total routes of Route Distinguisher(100:3): 1
 BGP routing table entry information of 1.1.5.1/32:
 Imported route. 
 Label information (Received/Applied): NULL/1028
 From: 0.0.0.0 (0.0.0.0)
 Route Duration: 00h05m45s  
 Direct Out-interface: GigabitEthernet0/0/2
 Original nexthop: 10.1.35.5
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <10.3.3.3 : 0>
 AS-path Nil, origin incomplete, MED 2, pref-val 0, valid, local, best, select, pre 255
 Advertised to such 1 peers:
    10.1.1.1
```

AR1 收到 AR3 通过 MP-BGP VPNv4 发布的路由 **`1.1.5.1/32`**，BGP 下一跳为 **`10.3.3.3`**。并且 AR1 向 CE1 的 Area 0 发布 Type 3 LSA，并设置 DN Bit。

```java{.line-numbers}
<AR1>display ip routing-table vpn-instance vpna 1.1.5.1
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  IBGP    255  2          RD   10.3.3.3        GigabitEthernet0/0/1
<AR1>display ospf 2 lsdb summary 1.1.5.1
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Sum-Net
  Ls id     : 1.1.5.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 1019 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 80000001 
  chksum    : 0x2e97
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Low
```

可以看到，AR4（CE1）收到了 AR1 发送过来的 **`1.1.5.1/32`** 的 Type3 LSA 并且带有 DN 标志位，因为 AR4 是普通的 CE1，因此可以使用带有 DN 标志位的 LSA 来计算并安装路由。但是 AR4 发送到 Area1 中的 **`1.1.5.1/32`** 的 Type3 LSA 就没有带有 DN 标志位了。核心原因是 OSPF 的 ABR 在跨区域传递路由时，并不是将收到的 Type 3 LSA 原封不动地转发到另一个区域，而是根据自身计算出的路由信息，重新生成一份 Type 3 LSA。新 LSA 的 DN Bit 不会自动继承原始 LSA 的 DN Bit。

```java{.line-numbers}
<AR4>display ospf 1 routing 

	 OSPF Process 1 with Router ID 4.4.4.4
		  Routing Tables 

 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 10.1.14.0/24       1     Transit    10.1.14.4       4.4.4.4         0.0.0.0
 10.1.34.0/24       1     Transit    10.1.34.4       4.4.4.4         0.0.0.1
 10.4.4.4/32        0     Stub       10.4.4.4        4.4.4.4         0.0.0.0
 1.1.5.1/32         3     Inter-area 10.1.14.1       10.1.1.1        0.0.0.0
 10.5.5.5/32        3     Inter-area 10.1.14.1       10.1.1.1        0.0.0.0
<AR4>display ospf 1 lsdb summary 1.1.5.1

	 OSPF Process 1 with Router ID 4.4.4.4
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Sum-Net
  Ls id     : 1.1.5.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 1137 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 80000001 
  chksum    : 0x2e97
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Medium
		         Area: 0.0.0.1
		 Link State Database 

  Type      : Sum-Net
  Ls id     : 1.1.5.1
  Adv rtr   : 4.4.4.4  
  Ls age    : 1136 
  Len       : 28 
  Options   :  E  
  seq#      : 80000001 
  chksum    : 0xb68b
  Net mask  : 255.255.255.255
  Tos 0  metric: 3
  Priority  : Low
```

AR3 收到 AR4 发送的这条 LSA（**`1.1.5.1/32`**）中没有 DN 标志位。也就是说 PE1 发布的 **`DN=1 Type 3 LSA`**，在普通 CE1（AR4）跨区域重新生成后，新的 Type 3 LSA 已经不再携带 DN Bit，并传递到了 PE2。但是由于 AR3 同时通过 Area 0 直接从 AR5 学到 **`1.1.5.1/32`**，它正常情况下会保留到 AR5 的区域内路径，而不会优选经 AR4 返回的区域间路径。

```java{.line-numbers}
<AR3>display ospf 2 lsdb summary 1.1.5.1

	 OSPF Process 2 with Router ID 10.3.3.3
		         Area: 0.0.0.0
		 Link State Database 

		         Area: 0.0.0.1
		 Link State Database 

  Type      : Sum-Net
  Ls id     : 1.1.5.1
  Adv rtr   : 4.4.4.4  
  Ls age    : 1549 
  Len       : 28 
  Options   :  E  
  seq#      : 80000001 
  chksum    : 0xb68b
  Net mask  : 255.255.255.255
  Tos 0  metric: 3
  Priority  : Medium
<AR3>display ip routing-table vpn-instance vpna 1.1.5.1
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  OSPF    10   1           D   10.1.35.5       GigabitEthernet0/0/2
```

### 7.9 特殊场景下的 DN Bit 处理与 Route-tag 防环

有些场景下需要关闭 DN 比特位的功能，否则会导致路由无法学习。比如当 PE 设备连接 MCE 设备场景时，MCE 设备需要计算 PE 设备发布的某些路由。如果 PE 设备发送的 LSA 的 DN 比特有置位，会导致 MCE 设备无法计算路由。

华为 VRP 系统禁用 DN 置位的命令是：**`dn-bit-set disable { summary | ase | nssa }`**。参数介绍：

- summary：指定不设置 Summary LSA 的 DN 位；
- ase：指定不设置 AS-external LSA 的 DN 位；
- nssa：指定不设置 NSSA LSA 的 DN 位。

但是在实际组网中，使用 **`dn-bit-set disable`** 命令用来禁止设置 OSPF LSA 的 DN 位，可能会产生路由环路。如果设置了 ase 参数或 nssa 参数，即对于 **`AS-external-LSA`** 或 NSSA LSA，这两种 LSA 也可以通过 Route-tag 命令配置相同的 Tag 值来防止路由环路。所以，**`dn-bit-set disable`** 命令只推荐在上面描述的场景中使用。

- **`dn-bit-set disable`** 命令仅支持在 OSPF 私网进程下配置，并且只在 PE 上生效。**`dn-bit-set disable`** **<font color="red">控制发送端，PE 在生成特定类型的 OSPF LSA 时，是否设置 DN Bit</font>**。
- 执行 **`dn-bit-check disable`** 命令可以控制对端 PE 路由器接收 OSPF 路由是否检查 DN 位。**`dn-bit-check disable`** **<font color="red">控制接收端 PE 收到带有 DN Bit 的 OSPF LSA 后，是否检查该标志并据此排除这条 LSA 的路由信息</font>**。

按照 IETF 的标准定义，在 CE 双归属环境中，通过在 LSA3 中设置 DN 位来避免潜在的路由环路，**在华为设备中，LSA5 和 LSA7 也可以通过另外的一种方式来阻止路由环路**。这两种 LSA（LSA5 和 LSA7）也可以使用 Route-tag 来防环。如果一台 PE 路由器设置了 Route-tag，当它将 OSPF 外部路由发布给 CE 之前会使路由带上 Route-tag。如果在同一站点的另一台 PE 路由器发现在接收到的 OSPF 外部路由中的 Route-tag 跟本端配置的一样，那么就会忽略该路由。如下所示。

<div align="center">
    <img src="mpls_static/66.png" width="450"/>
</div>

### 7.10 sham-link

