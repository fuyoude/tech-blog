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

**（3）import-route ospf 2**

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

通过 MPLS VPN 骨干网将运行 OSPF 协议的用户站点连接起来后，用户的业务通过骨干网进行传送。为增强网络的健壮性，现在很多大型的 MPLS VPN 场景中都会部署备份路径，这些备份路径我们称之为后门链路。通过后门链路将各站点的 OSPF 区域打通后，所有的 LSA 将会保持原样从一个站点传递到另一个站点，也就是说，**OSPF 内部路由（LSA1 和 LSA2）通过后门链路传递到远端的站点还是内部路由，而内部路由穿过 MPLS VPN 骨干传递过去后变成了区域间路由（LSA3）**，这就导致两个站点之间永远选择的是后门链路，而用户期望站点之间的数据流默认以 MPLS VPN 骨干为主路径，后门链路作备用路径。

<div align="center">
    <img src="mpls_static/67.png" width="750"/>
</div>

如上图所示，用户各站点都配置在相同的区域中，后门链路使得这些区域连接在一起，从而形成了一个区域。在上述 MPLS VPN Backbone 中底层通过公网 OSPF 1（Area 0）实现骨干网中各路由器的 loopback 接口互通，并在骨干区域中部署 MPLS LDP 建立公网标签转发路径（LSP）。控制平面在两台 PE 路由器 AR1 与 AR4 之间建立 MP-iBGP（AS 100）邻居，传递带标签的 VPNv4 私网路由。

AR1 和 AR4 面向客户的接口（**`G0/0/1`**）均绑定至私网实例 vpna，并通过运行在 vpna 内的 OSPF 2 与客户路由器 AR5、AR6 建立邻居，实现私网路由与 MP-BGP 的双向重分发。然后在客户站点 1（AR5）与站点 2（AR6）之间存在一条直连物理链路，同样运行 OSPF Area 0（**<font color="red">接口开销被刻意调大为 100</font>**），用于模拟企业原有的低速专线或备份链路。上述拓扑图的逻辑拓扑图如下所示：

<div align="center">
    <img src="mpls_static/68.png" width="700"/>
</div>

此时 AR1 和 AR4 之间没有配置 sham-link，导致 AR1 去往 **`1.1.6.1/32`** 路由的下一跳为 **`172.16.1.2`**，而不是 **`10.4.4.4`**。

sham-link 的 Cost 默认为 1，也通过设置 sham-link 的开销值来保证数据流走的是骨干网链路而非后门链路。sham-link 建立在两台 PE 路由器之间，**<font color="red">要求两台 PE 路由器分别在对应的 VRF 中建立一个 32 位掩码的环回接口来充当 sham-link 的端点，并且需要将这个地址宣告进 iBGP</font>**。LSA 在 sham-link 是不会周期性地泛洪的。

在区域视图下使用 sham-link 命令建立 sham-link，后面加上源和目标端点环回口地址。在 sham-link 建好后，注意，不能把 sham-link 的端点地址宣告进 VRF 下的 OSPF 进程，否则会导致 sham-link 抖动。

至此，总结一下在 PE 上部署 sham-link 的几点注意事项。

- 用于建立 sham-link 用的端点地址的掩码必须是 32 位的。
- 必须将端点地址宣告进 BGP，不能宣告到 VPN 实例的 OSPF 中。
- 用于 sham-link 端点的接口必须属于特定的 VRF。

#### 7.10.1 第一阶段实验配置

AR1 的配置如下所示：

```java{.line-numbers}
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
 ip address 10.1.12.1 255.255.255.0 
 mpls
 mpls ldp
#
interface GigabitEthernet0/0/1
 ip binding vpn-instance vpna
 ip address 172.16.1.1 255.255.255.0 
#
interface LoopBack0
 ip address 10.1.1.1 255.255.255.255 
#
bgp 100
 router-id 1.1.1.1
 undo default ipv4-unicast
 peer 10.4.4.4 as-number 100 
 peer 10.4.4.4 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  undo peer 10.4.4.4 enable
 # 
 ipv4-family vpnv4
  policy vpn-target
  peer 10.4.4.4 enable
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
 domain-id 0.0.0.10
 area 0.0.0.0 
  network 172.16.1.0 0.0.0.255 
```

AR4 的配置如下所示：

```java{.line-numbers}
#
 sysname AR4
#
ip vpn-instance vpna
 ipv4-family
  route-distinguisher 100:4
  vpn-target 1:1 export-extcommunity
  vpn-target 1:1 import-extcommunity
#
mpls lsr-id 10.4.4.4
mpls
#
mpls ldp
#
interface GigabitEthernet0/0/0
 ip address 10.1.34.4 255.255.255.0 
 mpls
 mpls ldp
#
interface GigabitEthernet0/0/1
 ip binding vpn-instance vpna
 ip address 172.16.2.1 255.255.255.0 
#
interface LoopBack0
 ip address 10.4.4.4 255.255.255.255 
#
bgp 100
 router-id 4.4.4.4
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
ospf 1 router-id 4.4.4.4 
 area 0.0.0.0 
  network 10.1.34.0 0.0.0.255 
  network 10.4.4.4 0.0.0.0 
#
ospf 2 router-id 10.4.4.4 vpn-instance vpna
 import-route bgp
 domain-id 0.0.0.10
 area 0.0.0.0 
  network 172.16.2.0 0.0.0.255 
```

#### 7.10.2 第一阶段实验结果分析

客户端的 AR5 与 AR6 之间存在一条直连后门链路，且两台设备与 AR1 均处于 OSPF Area 0 中。AR6 将自己的环回口地址 **`1.1.6.1/32`** 封装在 Type 1 Router-LSA 中向区域内泛洪，该 LSA 顺着后门链路经由 AR5 直接传递到了 AR1 的 ospf 2 进程中，如下所示。

```java{.line-numbers}
<AR1>display ospf 2 lsdb router 6.6.6.6
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Router
  Ls id     : 6.6.6.6
  Adv rtr   : 6.6.6.6  
  Ls age    : 1574 
  Len       : 60 
  Options   :  E  
  seq#      : 80000008 
  chksum    : 0x8eed
  Link count: 3
   * Link ID: 172.16.2.1   
     Data   : 172.16.2.2   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.56.6    
     Data   : 10.1.56.6    
     Link Type: TransNet     
     Metric : 100
   * Link ID: 1.1.6.1      
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium
```

AR1 的 ospf 2 收到该 LSA 后，通过 SPF 算法计算出一条到达 **`1.1.6.1/32`** 的区域内（Intra-Area）路由（下一跳为 AR5 的 **`172.16.1.2`**，总开销 Cost 为 **`101`**）。**<font color="red">由于 ospf 2 绑定了 **`vpn-instance vpna`**，这条计算出的路由直接被提交并安装进 vpna 的 IP 路由表中，协议类型为 OSPF，默认优先级为 10</font>**。至此，去往 **`1.1.6.1/32`** 的转发路径已经在 vpna IP 路由表中确立。

```java{.line-numbers}
<AR1>display ospf 2 routing 1.1.6.1
	 OSPF Process 2 with Router ID 10.1.1.1
 Destination : 1.1.6.1/32         
 AdverRouter : 6.6.6.6                  Area      : 0.0.0.0  
 Cost        : 101                      Type      : Stub 
 NextHop     : 172.16.1.2               Interface : GigabitEthernet0/0/1  
 Priority    : Medium                   Age       : 00h26m56s  
<AR1>display ip routing-table vpn-instance vpna 1.1.6.1
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.6.1/32  OSPF    10   101         D   172.16.1.2      GigabitEthernet0/0/1
```

与此同时，AR1 的 BGP 进程（**`ipv4-family vpn-instance vpna`**）通过两个不同的渠道，收集到了关于 **`1.1.6.1/32`** 的两条候选路由。第一个是由于 AR1 在 BGP 实例视图下配置了 **`import-route ospf 2`**，BGP 会扫描 vpna IP 路由表中处于活跃（Active）状态的 OSPF 路由，将刚安装进去的 **`1.1.6.1/32`** 提取到自己的 BGP 表中，如下所示，是 BGP vpna 路由表中关于 **`1.1.6.1/32`** 路由的详细信息。

```java{.line-numbers}
<AR1>display bgp vpnv4 vpn-instance vpna routing-table 1.1.6.1 32

 BGP local router ID : 1.1.1.1
 Local AS number : 100

 VPN-Instance vpna, Router ID 1.1.1.1:
 Paths:   2 available, 1 best, 1 select
 BGP routing table entry information of 1.1.6.1/32:
 Imported route. 
 From: 0.0.0.0 (0.0.0.0)
 Route Duration: 00h34m56s  
 Direct Out-interface: GigabitEthernet0/0/1
 Original nexthop: 172.16.1.2
 Qos information : 0x0
 Ext-Community:OSPF DOMAIN ID <0.0.0.10 : 0>, OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <10.1.1.1 : 0>
 AS-path Nil, origin incomplete, MED 102, pref-val 0, valid, local, best, select, pre 10
 Not advertised to any peer yet
```

第二个是直连 AR6 的对端 PE（AR4）同样通过 OSPF 学到了 **`1.1.6.1/32`**（Cost 为 1），并将其引入 BGP，附加 RD 和 **`Export RT 1:1`** 后，封装成 VPNv4 路由通过 MP-BGP 传给 AR1。AR1 收到后校验本地 **`Import RT 1:1`** 匹配成功，便将这条路径也收纳进 BGP 的 vpna 路由表中。

```java{.line-numbers}
 BGP routing table entry information of 1.1.6.1/32:
 Label information (Received/Applied): 1028/NULL
 From: 10.4.4.4 (4.4.4.4)
 Route Duration: 00h34m39s  
 Relay Tunnel Out-Interface: GigabitEthernet0/0/0
 Relay token: 0x5
 Original nexthop: 10.4.4.4
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.10 : 0>, OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <10.4.4.4 : 0>
 AS-path Nil, origin incomplete, MED 2, localpref 100, pref-val 0, valid, internal, pre 255, IGP cost 3, not preferred for route type
 Not advertised to any peer yet
```

面对 **`1.1.6.1/32`** 的两条有效路径，BGP 需要在自己的路由表内部执行选路决策。虽然从 AR4 学来的路由 MED 值仅为 2（远小于本地引入路由的 102），但依据华为设备的 BGP 选路原则，本地通过 **`import-route`** 引入的路由（下一跳为 **`0.0.0.0`**），**<font color="red">根据 BGP 选路原则的第三个原则，`import-route` 命令引入的路由要优于从对等体学习的路由</font>**。其路径类型优先级高于从 IBGP 对等体学习到的路由（下一跳为 **`10.4.4.4`**）。

因此，在比较到 MED 属性之前，选路就已经分出胜负，而 AR4 传来的远端路由因 **`not preferred for route type`**（路由类型不优）在 BGP 内部中直接落选，仅保留 * i 状态。

由于 OSPF 的协议优先级高于 BGP，因此，vpna IP 路由表继续保留唯一的活跃路由 **`1.1.6.1/32`**，也就是由 ospf 2 提供的后门链路路由。ospf 2 下的 **`import-route bgp`** 命令只会从 vpna IP 路由表中抓取处于活跃（Active）状态的 BGP 路由进行重分发。由于当前 vpna IP 路由表中 **`1.1.6.1/32`** 的协议类型是 OSPF，不存在活跃的 BGP 路由，**`import-route bgp`** 无路由可引，自然也就不会向 AR5 泛洪对应的 Type 3 LSA。

AR1 访问 **`1.1.6.1/32`** 的业务流量，最终只能沿着开销高达 101 的低速后门链路转发，完全绕开了高速的 MPLS 骨干网。这也正是后续需要引入 Sham-Link（伪连接）来重构 OSPF 逻辑拓扑的根本原因。

#### 7.10.3 新增 sham-link 配置

为解决这个问题，引入了一种 OSPF 的特殊链路——sham-link。**<font color="red">sham-link 就像一条正常的 OSPF 链路一样，有自己的接口，可以在其上建立 OSPF 邻居关系，能够传递 OSPF 中所有报文，同步 LSDB 等，但是不能传递数据报文</font>**。也就是说 sham-link 可以传递 OSPF 的 5 种控制报文（hello、DD、LSR、LSU、LSAck），但是数据报文的发送不走 sham-link，而是走底层的 MPLS VPN 标签隧道。

并不是所有 Site 之间都需要配置 sham-link，OSPF 伪连接仅应用在属于同一个 OSPF 区域的两个 Site 之间存在后门链路的情况。如果 Site 之间没有后门链路，则不需要配置 OSPF 伪连接。如果 Site 不属于同一区域的情况，则不需要配置 OSPF 伪链路。

sham-link 可以被视为一个无编号、点到点的链路，它也会参与 SPF 的路由计算。**<font color="red">当 LSA 在伪链路中进行泛洪时，所有的 OSPF 路由类型都不会改变，不会转换成 LSA3 或者 LSA5 的类型</font>**。如果 sham-link 断掉，那么又恢复成前面所讲的机制。

在 AR1 和 AR4 之间创建 sham-link 的过程分为 3 步：

- 在两个 PE 的 vpna 中创建 Sham-Link 端点 Loopback1。
- 通过 MP-BGP VPNv4 交换两个端点的 **`/32`** 路由。
- 在两个 PE 的 **`OSPF 2 Area 0`** 中建立 Sham-Link，Cost 设置为 1。

首先在 AR1 和 AR4 上做的配置如下，首先创建 loopback1，然后将其绑定到各自的 vpna 实例上，然后将 AR1/AR4 上的 sham-link 端点地址 **`192.168.100.1`** 和 **`192.168.100.4`** 宣告到 BGP 中。

```java{.line-numbers}
// AR1 sham-link 配置
system-view

interface LoopBack1
 ip binding vpn-instance vpna
 ip address 192.168.100.1 255.255.255.255
quit

bgp 100
 ipv4-family vpn-instance vpna
  network 192.168.100.1 255.255.255.255
 quit
quit

return
// AR4 sham-link 配置
system-view

interface LoopBack1
 ip binding vpn-instance vpna
 ip address 192.168.100.4 255.255.255.255
quit

bgp 100
 ipv4-family vpn-instance vpna
  network 192.168.100.4 255.255.255.255
 quit
quit

return
```

在未配置 Sham-Link 时，客户路由穿过 MPLS VPN 骨干网后会退化为 **`Type 3 LSA`**，而经过客户后门链路传递的路由始终保持 **`Type 1/2 LSA`**（区域内路由）。依据 OSPF 选路原则（区域内路由优先于区域间路由），流量必然选择后门链路。

Sham-Link 的核心作用，**<font color="red">就是在两台 PE（AR1 与 AR4）之间建立一条跨越 MPLS 骨干网的逻辑链路，将 AR1、AR4、AR5、AR6 四台设备重新融合进同一张 OSPF Area 0 的区域内拓扑图中</font>**。因此 sham-link 必须与 PE-CE 之间的接口处于同一个 OSPF 进程（如 ospf 2）及同一个区域（Area 0），才能共享同一张 LSDB（链路状态数据库）参与 SPF 计算。虽然 PE1 侧运行 ospf 2、CE1 侧运行 ospf 1（进程号仅具本地意义），但只要均归属 Area 0，**`Type 1/2 LSA`** 就能通过 sham-link 穿过 MPLS 骨干网，不再降级为 Type 3 LSA。

sham-link 并非真实的物理网线，而是运行在 vpna 实例内的逻辑隧道。AR1 与 AR4 之间交互的 OSPF 协议报文（Hello、DD、LSU 等），本质上是以两个端点地址（**`192.168.100.1/32`** 与 **`192.168.100.4/32`**）为源、目的 IP 的私网单播报文。

由于运营商骨干网中的 P 路由器（AR2、AR3）仅运行公网协议与 LDP，完全不感知 vpna 私网路由，无法直接根据这两个私网 IP 转发报文。因此，必须将这两个端点地址宣告进 BGP 的 **`ipv4-family vpn-instance vpna`** 视图中。AR1 与 AR4 通过 MP-iBGP 互换携带 RD、RT 及内层私网标签的 VPNv4 路由，并将对端的端点路由安装进本地的 vpna IP 路由表（下一跳指向对端 PE 的公网 LoopBack0）。

当 PE1 通过 sham-link 发送 OSPF 单播报文到 PE2 时，**<font color="blue">PE1 会查询 vpna 路由表（目的地址为 PE2 的 **`192.168.100.4`** 单播地址），自动为其压入双层 MPLS 标签，使 OSPF 控制报文像普通 VPN 业务流量一样直接穿过 MPLS 骨干网送达对端 PE</font>**，从而建立起 Full 状态的邻接关系。

接着就是在 AR1 和 AR4 上创建 sham-link，从这里可以看出，因为 sham-link 归属于 ospf 2，同时 ospf 2 也绑定了 vpna 实例，所以 PE1 通过 sham-link 发送 OSPF 单播报文到 PE2 时，会去查询 vpna 的 IP 路由表。

```java{.line-numbers}
// AR1
ospf 2
 area 0.0.0.0
  sham-link 192.168.100.1 192.168.100.4 cost 1
 quit
quit
// AR4
ospf 2
 area 0.0.0.0
  sham-link 192.168.100.4 192.168.100.1 cost 1
 quit
quit
```

#### 7.10.4 sham-link 实验控制层面

**（1）OSPF 区域内拓扑收敛**

在 AR1 与 AR4 之间建立 Sham-Link 后，双方在 ospf 2（Area 0）中形成了一条逻辑点到点（P2P）链路。sham-link 创建后，在 **`router 10.1.1.1`** 与 **`router 10.4.4.4`** 各自的 Type 1 Router-LSA 中，均新增了一条指向对端的 P2P 链路描述。

针对 AR1 的 **`G0/0/0`** 和 **`G0/0/1`** 进行抓包，可以看到 AR6 通过 **`AR6-AR5-AR1`** 传递过来的 **`Router 6.6.6.6 Type1 LSA`**。但是没有收到 AR6 通过 **`AR6-AR4-sham-link-AR1`** 传递过来的 **`Router 6.6.6.6 Type1 LSA`**。

```java{.line-numbers}
Frame 19: 214 bytes on wire (1712 bits), 214 bytes captured (1712 bits) on interface -, id 0
Ethernet II, Src: HuaweiTe_d5:70:29 (00:e0:fc:d5:70:29), Dst: HuaweiTe_87:73:8c (00:e0:fc:87:73:8c)
Internet Protocol Version 4, Src: 172.16.1.2, Dst: 172.16.1.1
Open Shortest Path First
    OSPF Header
        Version: 2
        Message Type: LS Update (4)
        Packet Length: 180
        Source OSPF Router: 5.5.5.5
        Area ID: 0.0.0.0 (Backbone)
        Checksum: 0x38ed [correct]
        Auth Type: Null (0)
        Auth Data (none): 0000000000000000
    LS Update Packet
        Number of LSAs: 3
        LSA-type 1 (Router-LSA), len 60
        LSA-type 1 (Router-LSA), len 60
            .000 0000 0000 0011 = LS Age (seconds): 3
            0... .... .... .... = Do Not Age Flag: 0
            Options: 0x02, (E) External Routing
            LS Type: Router-LSA (1)
            Link State ID: 6.6.6.6
            Advertising Router: 6.6.6.6
            Sequence Number: 0x80000006
            Checksum: 0x350a
            Length: 60
            Flags: 0x00
            Number of Links: 3
            Type: Stub     ID: 172.16.2.0      Data: 255.255.255.0   Metric: 1
            Type: Transit  ID: 10.1.56.6       Data: 10.1.56.6       Metric: 100
            Type: Stub     ID: 1.1.6.1         Data: 255.255.255.255 Metric: 0
        LSA-type 2 (Network-LSA), len 32
```

以下 Frame 124 是 AR4 发送给 AR1 的 DD 报文，包含 **`6.6.6.6 Router LSA`**，Frame 123 是 AR1 发送给 AR4 的 DD 报文，也包含同样的 **`6.6.6.6 Router LSA`**。OSPF 建立邻接关系初期，双方通过 DB Description (DD) 报文交换数据库摘要，而非直接泛洪完整 LSA（Body）。DD 报文先告诉对端我这里有哪些 LSA、版本，从而让对端能够判断哪些 LSA 是一致的，哪些 LSA 缺失，哪些 LSA 版本较旧。而真正缺失或需要更新的完整 LSA，则由后续的 LSR/LSU 机制来请求和传送。

因此 AR1 通过后门链路（AR5）已经拥有了 **`6.6.6.6 Router-LSA`** 的完整实例，它在收到 AR4 的 DD 摘要时，会发现本地 LSDB 中已存在相同版本。由于本地已有，AR1 不会向 AR4 发送 LSR 请求该 LSA。因为没有请求，AR4 自然不会发送对应的 LSU。

```java{.line-numbers}
Frame 124: 250 bytes on wire (2000 bits), 250 bytes captured (2000 bits) on interface -, id 0
Ethernet II, Src: HuaweiTe_01:4e:d3 (00:e0:fc:01:4e:d3), Dst: HuaweiTe_87:73:8b (00:e0:fc:87:73:8b)
MultiProtocol Label Switching Header, Label: 1027, Exp: 6, S: 1, TTL: 253
Internet Protocol Version 4, Src: 192.168.100.4, Dst: 192.168.100.1
Open Shortest Path First
    OSPF Header
        Version: 2
        Message Type: DB Description (2)
        Packet Length: 212
        Source OSPF Router: 10.4.4.4
        Area ID: 0.0.0.0 (Backbone)
        Checksum: 0x638b [correct]
        Auth Type: Null (0)
        Auth Data (none): 0000000000000000
    OSPF DB Description
    // 省略其余 lsa 摘要
    LSA-type 1 (Router-LSA), len 60
        .000 0000 0001 1000 = LS Age (seconds): 24
        0... .... .... .... = Do Not Age Flag: 0
        Options: 0x02, (E) External Routing
        LS Type: Router-LSA (1)
        Link State ID: 6.6.6.6
        Advertising Router: 6.6.6.6
        Sequence Number: 0x80000009
        Checksum: 0x8cee
        Length: 60

Frame 123: 254 bytes on wire (2032 bits), 254 bytes captured (2032 bits) on interface -, id 0
Ethernet II, Src: HuaweiTe_87:73:8b (00:e0:fc:87:73:8b), Dst: HuaweiTe_01:4e:d3 (00:e0:fc:01:4e:d3)
MultiProtocol Label Switching Header, Label: 1025, Exp: 6, S: 0, TTL: 255
MultiProtocol Label Switching Header, Label: 1027, Exp: 6, S: 1, TTL: 255
Internet Protocol Version 4, Src: 192.168.100.1, Dst: 192.168.100.4
Open Shortest Path First
    OSPF Header
        Version: 2
        Message Type: DB Description (2)
        Packet Length: 212
        Source OSPF Router: 10.1.1.1
        Area ID: 0.0.0.0 (Backbone)
        Checksum: 0x6692 [correct]
        Auth Type: Null (0)
        Auth Data (none): 0000000000000000
    OSPF DB Description
    // 省略其余 lsa 摘要
    LSA-type 1 (Router-LSA), len 60
        .000 0000 0001 1001 = LS Age (seconds): 25
        0... .... .... .... = Do Not Age Flag: 0
        Options: 0x02, (E) External Routing
        LS Type: Router-LSA (1)
        Link State ID: 6.6.6.6
        Advertising Router: 6.6.6.6
        Sequence Number: 0x80000009
        Checksum: 0x8cee
        Length: 60
```

即使 AR1 没有从 AR4 收到 **`6.6.6.6 Router-LSA`**，AR1 也可以优选 Sham-Link，**<font color="red">这是因为 OSPF 选路依据的是整个 Area 的 LSDB 所描述的网络拓扑，而不是某条 LSA 从哪个邻居接收</font>**。在建立 sham-link 之前，AR1 的 LSDB 已经拥有四台路由器的 Router-LSA 和三个传输网络的 Network-LSA。因此，AR1 到达 **`1.1.6.1/32`** 的区域内路径为 **`AR1->AR5->AR6->Loopback0`**，cost 为 101。

<div align="center">
    <img src="mpls_static/69.png" width="450"/>
</div>

在 sham-link 建立好之后，AR1 和 AR4 的 OSPF 邻接关系达到 Full，两台 PE 在各自的 Router-LSA 中增加指向对方的 P2P 链路信息。AR6 的 Router-LSA 完全不需要发生变化。**<font color="red">因此 Sham-Link 的本质不是为 **`1.1.6.1/32`** 新增一条 LSA，而是为整个 Area 0 的拓扑新增一条可供 SPF 使用的逻辑链路</font>**，如下所示。

<div align="center">
    <img src="mpls_static/70.png" width="450"/>
</div>

**（2）OSPF 区域内路径计算**

sham-link 建立后，AR1 首先查看自己的 Router-LSA。从第 2 个 Link 可知，AR1 可以通过一条 cost 为 1 的 P2P 路径到达 AR4。

```java{.line-numbers}
<AR1>display ospf 2 lsdb router 10.1.1.1

	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Router
  Ls id     : 10.1.1.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 542 
  Len       : 48 
  Options   :  ASBR  ABR  E  
  seq#      : 80000006 
  chksum    : 0xfa92
  Link count: 2
   * Link ID: 172.16.1.1   
     Data   : 172.16.1.1   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.4.4.4     
     Data   : 0.127.128.1  
     Link Type: P-2-P        
     Metric : 1
```

接下来，AR1 在自己的 LSDB 中查找 **`10.4.4.4`** 的 Router-LSA，下面的输出有 2 个关键信息，首先 AR4 存在一条指回 AR1 的点到点链路，同时 AR4 连接着 **`172.16.2.0/24`** 传输网络。

```java{.line-numbers}
<AR1>display ospf 2 lsdb router 10.4.4.4
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Router
  Ls id     : 10.4.4.4
  Adv rtr   : 10.4.4.4  
  Ls age    : 597 
  Len       : 48 
  Options   :  ASBR  ABR  E  
  seq#      : 80000006 
  chksum    : 0xb0d1
  Link count: 2
   * Link ID: 172.16.2.1   
     Data   : 172.16.2.1   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.1.1     
     Data   : 0.127.128.1  
     Link Type: P-2-P        
     Metric : 1
```

接下来，AR1 读取 **`172.16.2.1`** 的 network LSA，这份 network-lsa 告诉 AR1 网络 **`172.16.2.0/24`** 连接了 AR4 和 AR6。这时 AR1 已经能够拼凑出路径 **`AR1-Sham-Link-AR4-172.16.2.0/24-AR6`**。

```java{.line-numbers}
<AR1>display ospf 2 lsdb network 172.16.2.1

	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Network
  Ls id     : 172.16.2.1
  Adv rtr   : 10.4.4.4  
  Ls age    : 1024 
  Len       : 32 
  Options   :  E  
  seq#      : 80000007 
  chksum    : 0x14f
  Net mask  : 255.255.255.0
  Priority  : Low
     Attached Router    10.4.4.4
     Attached Router    6.6.6.6
```

最后，读取就是配置 sham-link 之后由 AR5 发送给 AR1 的 Router LSA。因此 AR1 计算出通过 sham-link 去往 **`1.1.6.1/32`** 的路径的 cost 为 2，**`AR1-AR4-AR6`** 为最优路径。从 AR1 的 **`G0/0/1`** 抓包结果可知，该 router LSA 是由 AR5 发送给 AR1 的。 

```java{.line-numbers}
<AR1>display ospf 2 lsdb router 6.6.6.6
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Router
  Ls id     : 6.6.6.6
  Adv rtr   : 6.6.6.6  
  Ls age    : 850 
  Len       : 60 
  Options   :  E  
  seq#      : 80000009 
  chksum    : 0x8cee
  Link count: 3
   * Link ID: 172.16.2.1   
     Data   : 172.16.2.2   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.56.6    
     Data   : 10.1.56.6    
     Link Type: TransNet     
     Metric : 100
   * Link ID: 1.1.6.1      
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium

Frame 37: 158 bytes on wire (1264 bits), 158 bytes captured (1264 bits) on interface -, id 0
Ethernet II, Src: HuaweiTe_d5:70:29 (00:e0:fc:d5:70:29), Dst: IPv4mcast_05 (01:00:5e:00:00:05)
Internet Protocol Version 4, Src: 172.16.1.2, Dst: 224.0.0.5
Open Shortest Path First
    OSPF Header
    LS Update Packet
        Number of LSAs: 2
        LSA-type 1 (Router-LSA), len 36
        LSA-type 1 (Router-LSA), len 60
            .000 0000 0000 0010 = LS Age (seconds): 2
            0... .... .... .... = Do Not Age Flag: 0
            Options: 0x02, (E) External Routing
            LS Type: Router-LSA (1)
            Link State ID: 6.6.6.6
            Advertising Router: 6.6.6.6
            Sequence Number: 0x80000009
            Checksum: 0x8cee
            Length: 60
            Flags: 0x00
            Number of Links: 3
            Type: Transit  ID: 172.16.2.1      Data: 172.16.2.2      Metric: 1
            Type: Transit  ID: 10.1.56.6       Data: 10.1.56.6       Metric: 100
            Type: Stub     ID: 1.1.6.1         Data: 255.255.255.255 Metric: 0
```

**（3）OSPF 路由抑制**

依据 RFC 4577 规范，sham-link 仅作为承载 OSPF 控制平面协议报文的逻辑链路，不直接承担业务数据转发。**<font color="red">当某条目的网段的最优 OSPF 路径下一跳为 sham-link 时，实际数据转发必须交由对应的 BGP VPNv4 路由执行，且该 OSPF 路由不得再次被重分发回 BGP</font>**。

为满足上述协议要求，华为设备采取了直接抑制机制：**<font color="red">凡是 SPF 计算出的最优下一跳为 sham-link 的路由，均不再写入 OSPF 协议路由表（**`display ospf 2 routing`**），也不再下发至 vpna IP 路由表</font>**。因此，**`1.1.6.1/32`** 从 AR1 的 ospf 2 路由表中消失。

```java{.line-numbers}
<AR1>display ospf 2 routing 
	 OSPF Process 2 with Router ID 10.1.1.1
		  Routing Tables 
 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 172.16.1.0/24      1     Transit    172.16.1.1      10.1.1.1        0.0.0.0
 1.1.5.1/32         1     Stub       172.16.1.2      5.5.5.5         0.0.0.0
 10.1.56.0/24       101   Transit    172.16.1.2      6.6.6.6         0.0.0.0
 Total Nets: 3  
 Intra Area: 3  Inter Area: 0  ASE: 0  NSSA: 0 
```

以 AR6 的 Router-LSA 中描述的传输网络 **`10.1.56.0/24`** 为例，进一步验证 OSPF 路由抑制机制。

初始状态下，AR1 经 Sham-Link 到达该网段的 Cost 为 102（**`1 + 1 + 100`**），经 AR5 到达该网段的 Cost 为 101（**`1 + 100`**）。因此，SPF 优选经 AR5 的区域内路径，下一跳为 **`172.16.1.2`**，而非 Sham-Link，该路由正常保留在 OSPF 2 路由表中。将 AR1 的 **`GE0/0/1`** 和 AR5 的 **`GE0/0/0`** 接口的 OSPF Cost 均修改为 10 后，经 AR5 的路径开销增至 110（**`10 + 100`**），而经 Sham-Link 的路径开销仍为 102。此时，SPF 优选经 Sham-Link 到达该网段的路径，触发 OSPF 路由抑制机制，导致 **`10.1.56.0/24`** 从 OSPF 2 路由表中消失，实验回显如下。

```java{.line-numbers}
[AR1-GigabitEthernet0/0/1]display ospf 2 routing 
	 OSPF Process 2 with Router ID 10.1.1.1
		  Routing Tables 
 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 172.16.1.0/24      10    Transit    172.16.1.1      10.1.1.1        0.0.0.0
 1.1.5.1/32         10    Stub       172.16.1.2      5.5.5.5         0.0.0.0
 Total Nets: 2  
 Intra Area: 2  Inter Area: 0  ASE: 0  NSSA: 0 
```

**（4）BGP vpna 路由表与 vpna IP 路由表的联动更新**

由于 vpna IP 路由表中不再存在 **`1.1.6.1/32`** 的活跃 OSPF 路由，AR1 的 BGP 进程所配置的 **`import-route ospf 2`** 随即撤销了此前本地引入的路由。

```java{.line-numbers}
<AR1>display bgp vpnv4 vpn-instance vpna routing-table 1.1.6.1

 BGP local router ID : 1.1.1.1
 Local AS number : 100

 VPN-Instance vpna, Router ID 1.1.1.1:
 Paths:   1 available, 1 best, 1 select
 BGP routing table entry information of 1.1.6.1/32:
 Label information (Received/Applied): 1029/NULL
 From: 10.4.4.4 (4.4.4.4)
 Route Duration: 00h17m26s  
 Relay Tunnel Out-Interface: GigabitEthernet0/0/0
 Relay token: 0x5
 Original nexthop: 10.4.4.4
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.10 : 0>, OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <10.4.4.4 : 0>
 AS-path Nil, origin incomplete, MED 2, localpref 100, pref-val 0, valid, internal, best, select, active, pre 255, IGP cost 3
 Not advertised to any peer yet

<AR1>display ip routing-table vpn-instance vpna 1.1.6.1
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.6.1/32  IBGP    255  2          RD   10.4.4.4        GigabitEthernet0/0/0
```

此时，AR1 的 BGP vpna 路由表中仅保留由远端 PE（AR4）通过 MP-iBGP 传递的 VPNv4 路由（下一跳为 **`10.4.4.4`**）。该路由成为 BGP 最优路径（**`best, select, active`**），并顺利安装进 vpna IP 路由表中。

**（5）触发 `import-route bgp` 生成 Type 3 LSA**

当来自 AR4 的 **`1.1.6.1/32`** 路由成为 vpna IP 路由表中的活跃（Active）IBGP 路由后，AR1 在 ospf 2 下配置的 **`import-route bgp`** 终于在 vpna ip 路由表中匹配到了活跃的 BGP 路由。又因为 AR1 与 AR4 配置了相同的 **`domain-id 0.0.0.10`**，且该路由原始属性为 OSPF 区域内路由，AR1 便在 ospf 2 的 Area 0 中为其生成了一条带有 DN 防环置位的 Type 3 Summary-LSA。

```java{.line-numbers}
<AR1>display ospf 2 lsdb summary 1.1.6.1
	 OSPF Process 2 with Router ID 10.1.1.1
		         Area: 0.0.0.0
		 Link State Database 
  Type      : Sum-Net
  Ls id     : 1.1.6.1
  Adv rtr   : 10.1.1.1  
  Ls age    : 1077 
  Len       : 28 
  Options   :  E  DN  
  seq#      : 80000001 
  chksum    : 0x23a1
  Net mask  : 255.255.255.255
  Tos 0  metric: 2
  Priority  : Low
```

#### 7.10.5 sham-link 实验数据层面

接下来分析，数据报文从 AR1 到 AR6 的转发过程。首先 AR1 收到来自私网的 ICMP 请求报文，首先查询 vpna 的路由表。发现目的地址 **`1.1.6.1/32`** 的下一跳为 AR4。AR1 对该数据报文贴上双层标签，内层私网标签为 1029。外层公网标签根据 LDP LSP 表项，查找到达 **`10.4.4.4`** 的公网隧道标签为 1025，将其压入栈顶。

```java{.line-numbers}
<AR1>display mpls lsp        
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name                                           
10.4.4.4/32        NULL/1025     -/GE0/0/0                                     
10.4.4.4/32        1026/1025     -/GE0/0/0                                      
```

AR2 作为骨干网的中间路由器，仅关注外层标签 1025。AR2 查询本地 MPLS 转发表，发现该标签对应的出标签为 1024。AR2 执行 Swap 操作，将 1025 替换为 1024，并转发给 AR3。AR3 作为到达 AR4 的倒数第二跳，执行 PHP 操作。AR3 直接将外层标签 1024 弹出，报文只携带私网标签 1029 被发送给 AR4。

```java{.line-numbers}
<AR2>display mpls lsp
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name                                        
10.4.4.4/32        NULL/1024     -/GE0/0/1                                     
10.4.4.4/32        1025/1024     -/GE0/0/1 

<AR3>display mpls lsp 
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name                                           
10.4.4.4/32        NULL/3        -/GE0/0/1                                     
10.4.4.4/32        1024/3        -/GE0/0/1  
```

AR4 收到 AR3 转发过来的仅携带私网标签 1029 的 MPLS 报文。由于本实验采用 Per-route（每路由每标签）的 VPN 标签分配方式，私网标签 1029 对应的是 vpna 实例中的 **`1.1.6.1/32`** 路由。AR4 根据入标签 1029 查询本地 MPLS 标签转发表，获得对应的标签操作、出接口 **`G0/0/1`** 以及下一跳 **`172.16.2.2`**（AR6）。随后，AR4 执行 Pop 操作，移除私网标签 1029，然后直接将报文转发给 AR6，无需再次查询 vpna 的 IP 转发表。