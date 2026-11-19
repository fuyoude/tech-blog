# MPLS VPN 基础

## 5.CE-PE 之间运行静态路由

### 5.1 网络拓扑介绍

如下图所示，某客户（VPN A）的两个站点（Site1 和 Site2）通过 MPLS VPN 实现互访，其中 CE1 连接 Site1，CE1 通过服务提供商的 PE1 接入至 MPLS VPN 骨干网。CE2 连接 Site2，CE2 通过服务提供商的 PE2 接入至 MPLS VPN 骨干网。

<div align="center">
    <img src="mpls_static/49.png" width="750"/>
</div>

服务商骨干网的 BGP AS 号为 100，IGP 使用 OSPF，服务商为客户分配的 VPN 实例名为 vpna，RD 值为 **`100:1`**，Site1 和 Site2 的 Import RT 值和 Export RT 值均为 **`1:1`**，PE-CE 之间使用静态路由。

在上述拓扑图中，作为经典的 MPLS L3 VPN 技术中，OSPF 作为运营商骨干网的 IGP，提供公网路由可达性，确保各 PE、P 路由器之间能够互相到达。**<font color="red">MPLS LDP 基于骨干网的 IP 可达性分发公网标签，建立从入口 PE 到出口 PE 的公网 LSP。MP-BGP VPNv4 在 PE 路由器之间交换客户 VPN 路由和私网标签</font>**，私网标签由通告 VPN 路由的出口 PE 分配，再通过 MP-BGP 随 VPNv4 路由传递给入口 PE。

中间 P 路由器 根据外层 MPLS 标签完成标签交换，无需学习客户 VPN 路由，不需要运行 BGP。出口 PE 根据私网标签确定对应 VRF 中的转发信息，并将数据包发送给目标 CE，最终到达目的站点。上述拓扑图的逻辑拓扑图如下所示。

<div align="center">
    <img src="mpls_static/50.png" width="650"/>
</div>

具体 AR1 上的配置如下所示：

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
 ip address 172.16.1.1 255.255.255.0 
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
  import-route static
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 10.1.1.1 0.0.0.0 
  network 10.1.12.0 0.0.0.255 
#
ip route-static vpn-instance vpna 1.1.5.1 255.255.255.255 172.16.1.2
```

AR4 上的配置如下所示：

```java{.line-numbers}
#
 sysname AR4
#
ip vpn-instance vpna
 ipv4-family
  route-distinguisher 100:1
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
 router-id 10.4.4.4
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
  import-route static
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.0 
  network 10.1.34.0 0.0.0.255 
  network 10.4.4.4 0.0.0.0 
#
ip route-static vpn-instance vpna 1.1.6.1 255.255.255.255 172.16.2.2
```

注意，在 PE1 的上述配置中，**`ipv4-family vpn-instance vpna import-route static`** **<font color="red">用来将 VPN 实例 vpna 中的静态路由引入到 BGP 中</font>**。而 **`ipv4-family vpnv4 peer 10.1.1.1 enable`** 表示允许 PE2 与 PE1 通过 MP-BGP 交换 VPNv4 地址族的路由信息。因此综合上述配置，效果就是使得该 vpna 中的静态路由能够通过 MP-BGP VPNv4 发布给远端 PE。

**`ip route-static vpn-instance vpna 1.1.5.1 255.255.255.255 172.16.1.2`** 表示在 PE1 的 vpna 实例中配置了一条静态路由，指向 CE1 的地址。

在配置完成之后，在 PE1 和 PE2 上查看 BGP VPNv4 邻居建立的情况，可以看到 PE1 和 PE2 之间的 BGP VPNv4 邻居已经建立成功，状态为 Established。

```java{.line-numbers}
<AR1>display bgp vpnv4 all peer 
 BGP local router ID : 10.1.1.1
 Local AS number : 100
 Total number of peers : 1		  Peers in established state : 1
  Peer            V          AS  MsgRcvd  MsgSent  OutQ  Up/Down       State Pre fRcv
  10.4.4.4        4         100     1351     1352     0 22:28:45 Established     1
<AR4>display bgp vpnv4 all peer 
 BGP local router ID : 10.4.4.4
 Local AS number : 100
 Total number of peers : 1		  Peers in established state : 1
  Peer            V          AS  MsgRcvd  MsgSent  OutQ  Up/Down       State Pre fRcv
  10.1.1.1        4         100     1352     1352     0 22:29:30 Established     1
```

### 5.2 控制层面

这里以 Site2 中的 **`1.1.6.1/32`** 路由到达 Site1 为例，分析 PE–CE 之间采用静态路由时，MPLS L3VPN 控制平面的路由传递过程。

需要注意，**<font color="red">与 OSPF 实验不同，静态路由实验中的 CE 不会主动向 PE 发布路由，而是通过在 PE1 和 PE2 上手工配置</font>**，使 PE1 获得本地客户 CE1 网络 **`1.1.5.1/32`** 路由信息，使 PE2 获得本地客户 CE2 网络 **`1.1.6.1/32`** 路由信息。然后 PE1 通过 MP-BGP VPNv4 获取远端 CE2 的客户路由 **`1.1.6.1/32`** 信息，PE2 也是通过 MP-BGP VPNv4 获取远端 CE1 的客户路由 **`1.1.5.1/32`** 信息。

```java{.line-numbers}
[AR1]display ip routing-table vpn-instance vpna
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: vpna
         Destinations : 6        Routes : 6        
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  Static  60   0          RD   172.16.1.2      GigabitEthernet0/0/0
        1.1.6.1/32  IBGP    255  0          RD   10.4.4.4        GigabitEthernet0/0/1
<AR4>display ip routing-table vpn-instance vpna
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: vpna
         Destinations : 6        Routes : 6        
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  IBGP    255  0          RD   10.1.1.1        GigabitEthernet0/0/0
        1.1.6.1/32  Static  60   0          RD   172.16.2.2      GigabitEthernet0/0/1
```

在 CE1 上配置访问 Site2 的静态路由，下一跳为 PE1 的接口地址。在 CE2 上配置访问 Site1 的静态路由，下一跳为 PE2 的接口地址。如下所示：

```java{.line-numbers}
[AR5]display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 9        Routes : 9        
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  Direct  0    0           D   127.0.0.1       LoopBack0
        1.1.6.1/32  Static  60   0          RD   172.16.1.1      GigabitEthernet0/0/0
<AR6>display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 9        Routes : 9        
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        1.1.5.1/32  Static  60   0          RD   172.16.2.1      GigabitEthernet0/0/0
        1.1.6.1/32  Direct  0    0           D   127.0.0.1       LoopBack0
```

**第 1 步是 PE2 通过静态路由获取 Site2 的客户路由** **`1.1.6.1/32`**。在 PE2（AR4）上配置一条指向 CE2 的 VPN 静态路由 **`ip route-static vpn-instance vpna 1.1.6.1 255.255.255.255 172.16.2.2`**。该命令将目的网络为 **`1.1.6.1/32`**、下一跳为 CE2 接口地址 **`172.16.2.2`** 的静态路由添加到 VPN 实例 vpna 中，使 PE2 具备访问 Site2 客户网络的路由。

**第 2 步是 PE2 将静态路由引入 MP-BGP，生成 VPNv4 路由**。PE2 在 BGP 进程 100 的 vpna IPv4 地址族中配置：

```java{.line-numbers}
bgp 100
 ipv4-family vpn-instance vpna
  import-route static
```

PE2 将静态路由 **`1.1.6.1/32`** 引入 BGP 后，根据 VPN 实例配置生成 VPNv4 路由，并为其附加 **`RD 100:1`**、**`Export RT 1:1`** 等属性。同时，PE2 为该 VPN 路由分配私网标签 1027，用于后续 MPLS VPN 数据转发。根据下列输出可以看出，**<font color="red">`imported route` 表示该 `1.1.6.1/32` 路由由其他路由来源引入 BGP</font>**，**<font color="red">`Advertised to such 1 peers: 10.1.1.1` 表示路由已经向 PE1 通告</font>**。

```java{.line-numbers}
<AR4>display bgp vpnv4 all routing-table 1.1.6.1 32
 BGP local router ID : 10.4.4.4
 Local AS number : 100

 Total routes of Route Distinguisher(100:1): 1
 BGP routing table entry information of 1.1.6.1/32:
 Imported route. 
 Label information (Received/Applied): NULL/1027
 From: 0.0.0.0 (0.0.0.0)
 Route Duration: 23h31m51s  
 Direct Out-interface: GigabitEthernet0/0/1
 Original nexthop: 172.16.2.2
 Qos information : 0x0
 Ext-Community:RT <1 : 1>
 AS-path Nil, origin incomplete, MED 0, pref-val 0, valid, local, best, select, pre 255
 Advertised to such 1 peers:
    10.1.1.1
```

**第 3 步是 PE2 通过 MP-iBGP 将 VPNv4 路由通告给 PE1**，PE1 和 PE2 均属于运营商 AS 100，通过 Loopback0 地址建立 MP-iBGP VPNv4 邻居关系。当 VPNv4 邻居处于 Established 状态，且路由符合发布条件时，PE2 通过 MP-BGP UPDATE 报文将 **`1.1.6.1/32`** 的 VPNv4 路由通告给 PE1。该路由携带 RD、RT、私网标签及 BGP 下一跳等信息。其中 BGP 下一跳可以在 AR1 上查看，即 PE2 的 Loopback0 地址 **`10.4.4.4`**。

```java{.line-numbers}
<AR1>display bgp vpnv4 all routing-table 1.1.6.1 32
 BGP local router ID : 10.1.1.1
 Local AS number : 100

 Total routes of Route Distinguisher(100:1): 1
 BGP routing table entry information of 1.1.6.1/32:
 Label information (Received/Applied): 1027/NULL
 From: 10.4.4.4 (10.4.4.4)
 Route Duration: 23h39m27s  
 Relay IP Nexthop: 10.1.12.2
 Relay IP Out-Interface: GigabitEthernet0/0/1
 Relay Tunnel Out-Interface: GigabitEthernet0/0/1
 Relay token: 0x5
 Original nexthop: 10.4.4.4
 Qos information : 0x0
 Ext-Community:RT <1 : 1>
 AS-path Nil, origin incomplete, MED 0, localpref 100, pref-val 0, valid, internal, best, select, pre 255, IGP cost 3
 Not advertised to any peer yet
```

需要注意，MP-BGP 负责在 PE 之间传播客户 VPN 路由，骨干网中的 P1、P2 不需要学习 **`1.1.6.1/32`** 这条客户路由。

**第 4 步是 PE1 根据 RT 匹配结果将 VPNv4 路由导入本地 VPN 实例**。PE1 收到 PE2 通告的 VPNv4 路由后，将路由携带的 **`Export RT 1:1`** 与本地 VPN 实例 vpna 的 **`Import RT 1:1`** 进行匹配。匹配成功后，PE1 将该 VPNv4 路由导入本地 vpna，形成一条目的网络为 **`1.1.6.1/32`** 的 IBGP 路由，并保留 PE2 通告的私网标签及相关转发信息。其中，**`Label: 1027`** 表示 PE2 分配并通告给 PE1 的私网标签。

```java{.line-numbers}
<AR1>display bgp vpnv4 all routing-table 1.1.6.1 32
 BGP local router ID : 10.1.1.1
 Local AS number : 100

 Total routes of Route Distinguisher(100:1): 1
 BGP routing table entry information of 1.1.6.1/32:
 Label information (Received/Applied): 1027/NULL
 From: 10.4.4.4 (10.4.4.4)
 Route Duration: 23h39m27s  
 Relay IP Nexthop: 10.1.12.2
 Relay IP Out-Interface: GigabitEthernet0/0/1
 Relay Tunnel Out-Interface: GigabitEthernet0/0/1
 Relay token: 0x5
 Original nexthop: 10.4.4.4
 Qos information : 0x0
 Ext-Community:RT <1 : 1>
 AS-path Nil, origin incomplete, MED 0, localpref 100, pref-val 0, valid, internal best, select, pre 255, IGP cost 3
 Not advertised to any peer yet
<AR1>display ip routing-table vpn-instance vpna 1.1.6.1 verbose 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : vpna
Summary Count : 1

Destination: 1.1.6.1/32
     Protocol: IBGP             Process ID: 0
   Preference: 255                    Cost: 0
      NextHop: 10.4.4.4          Neighbour: 10.4.4.4
        State: Active Adv Relied       Age: 1d00h10m27s
          Tag: 0                  Priority: low
        Label: 1027                QoSInfo: 0x0
   IndirectID: 0x3              
 RelayNextHop: 10.1.12.2         Interface: GigabitEthernet0/0/1
     TunnelID: 0x5                   Flags: RD
```

其中，**`10.4.4.4`** 是远端 PE2 的 Loopback0 地址，也是该 VPN 路由的 BGP 下一跳。PE1 根据骨干 OSPF 路由对其进行递归解析，确定实际转发下一跳为 P1（**`10.1.12.2`**），并关联到达 PE2（**`10.1.4.4`**）的公网 LDP LSP。

**第 5 步 CE1 通过静态路由获得访问 Site2 的路径**。由于 PE–CE 之间采用静态路由，PE1 不会通过动态路由协议向 CE1 通告远端客户路由。因此，需要在 CE1 上手工配置访问 Site2 的静态路由 **`ip route-static 1.1.6.1 255.255.255.255 172.16.1.1`**。

控制平面的 vpn 路由传递过程总结为：在 MPLS L3VPN 的静态路由实验中，由于 PE–CE 之间不运行动态路由协议，因此客户路由的获取依赖管理员手工配置。首先，PE2 在 VPN 实例 vpna 中配置指向 CE2 的静态路由 **`1.1.6.1/32`**，使 PE2 具备访问 Site2 客户网络的路由。随后，PE2 将该 VPN 实例 vpna 中的静态路由引入 BGP，并根据 VPN 实例配置生成携带 **`RD 100:1`**、**`Export RT 1:1`** 及私网标签的 VPNv4 路由，再通过 MP-iBGP 通告给 PE1。

PE1 收到该 VPNv4 路由后，将其 **`Export RT 1:1`** 与本地 VPN 实例 vpna 的 **`Import RT 1:1`** 进行匹配，匹配成功后将路由导入本地 VPN 实例 vpna 路由表，形成目的网络为 **`1.1.6.1/32`** 的 IBGP 路由，并保留 PE2 通告的私网标签 1027。该路由的 BGP 下一跳为 PE2（**`10.4.4.4`**），经过公网 OSPF 路由递归解析后，实际转发下一跳为 **`10.1.12.2`**。最后，由于 PE–CE 之间使用静态路由，CE1 需要手工配置到达 **`1.1.6.1/32`** 的静态路由。

### 5.3 数据层面

**<font color="red">由于运营商骨干网中的 P 路由器不维护客户 VPN 路由，因此无法直接根据客户 IP 地址完成跨站点转发，需要利用 MPLS 标签交换技术</font>**。业务报文进入 MPLS 骨干网时，入站 PE 通常会封装两层标签：外层公网标签用于引导报文沿 LDP LSP 到达远端 PE，内层私网标签用于出站 PE 识别对应的 VPN 转发信息。

这里以 Site1 中 CE1 的 Loopback0 地址 **`1.1.5.1`** 向 Site2 中 CE2 的 Loopback0 地址 **`1.1.6.1`** 发送业务数据为例，分析整个 MPLS L3VPN 数据平面的转发过程。

#### 5.3.1 数据由 CE1 转发给入站 PE1

当 CE1（AR5）需要向 Site2 的 **`1.1.6.1/32`** 发送数据时，首先查询本地 IPv4 路由表。由于 CE1 已配置静态路由 **`ip route-static 1.1.6.1 255.255.255.255 172.16.1.1`**，因此，CE1 将目的地址为 **`1.1.6.1`** 的普通 IPv4 报文转发给下一跳 PE1（**`172.16.1.1`**），此时数据包没有携带 MPLS 标签。

PE1（AR1）收到报文后，根据入接口 **`GE0/0/0`** 绑定的 VPN 实例 vpna 查询对应的 VPN 转发表，匹配到目的网络为 **`1.1.6.1/32`** 的 IBGP 路由，获取 PE2 通过 MP-BGP 通告的私网标签 1027，以及 BGP 下一跳 PE2 的 Loopback0 地址 **`10.4.4.4`**。

由于 BGP 下一跳 **`10.4.4.4`** 并非直连，PE1 需要使用公网标签及对应的 LDP LSP 隧道进行数据包的发送。通过如下命令可以找到以 PE2 Loopback0 地址 **`10.4.4.4/32`** 为 FEC 的公网 LDP LSP，从而获取公网出标签 1026 以及对应的标签操作 PUSH。

```java{.line-numbers}
<AR1>display mpls lsp
-------------------------------------------------------------------------------
                 LSP Information: BGP  LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
1.1.5.1/32         1027/NULL     -/-                            vpna           
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
10.2.2.2/32        NULL/3        -/GE0/0/1                                     
10.2.2.2/32        1024/3        -/GE0/0/1                                     
10.3.3.3/32        NULL/1024     -/GE0/0/1                                     
10.3.3.3/32        1025/1024     -/GE0/0/1                                     
10.1.1.1/32        3/NULL        -/-                                           
10.4.4.4/32        NULL/1026     -/GE0/0/1                                     
10.4.4.4/32        1026/1026     -/GE0/0/1   
<AR1>display mpls lsp include 10.4.4.4 32 verbose 
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
  No                  :  1
  VrfIndex            :        
  Fec                 :  10.4.4.4/32
  Nexthop             :  10.1.12.2
  In-Label            :  NULL
  Out-Label           :  1026
  In-Interface        :  ----------
  Out-Interface       :  GigabitEthernet0/0/1
  LspIndex            :  6149
  Token               :  0x5
  FrrToken            :  0x0
  LsrType             :  Ingress
  Outgoing token      :  0x0
  Label Operation     :  PUSH
  Mpls-Mtu            :  1500
  TimeStamp           :  89172sec
  Bfd-State           :  ---
  BGPKey              :  ------
```

其中，PE1 的 LDP LSP 表中 **`NULL/1026`** 表示 PE1 作为该公网 LSP 的入站 LSR，无须匹配公网入标签，而是使用出标签 1026 对业务报文执行 PUSH（压入标签）操作。这个标签由下游 P1 通过 LDP 通告给 PE1。与此同时，PE2 为本地 VPN 路由 **`1.1.6.1/32`** 分配的私网标签为 1028。

随后，PE1 将封装完成的 MPLS 报文通过 GE0/0/1 转发给 P1（10.1.12.2），使业务数据进入 MPLS 骨干网。

#### 5.3.2 数据由入站 PE1 经 P1、P2 转发至出站 PE2

当 PE1 将双层标签报文发送到 MPLS 骨干网后，中间的 P 路由器只需根据报文最外层的公网标签查找 MPLS 标签转发表，无需查询客户 VPN 路由，也不需要识别内层私网标签。

首先，P1（AR2）收到 PE1 发送的 MPLS 报文后，根据公网入标签 1026 查找 MPLS 标签转发表。

在 P1 上执行：

<AR2> display mpls lsp

根据此前的实际输出，P1 存在以下标签转发记录：

FEC             In/Out Label    In/Out IF
10.4.4.4/32     1026/1026       -/GE0/0/1

这表示 P1 收到公网标签为 1026 的报文后，执行 SWAP（标签交换） 操作，将入标签 1026 替换为出标签 1026，随后通过 GE0/0/1 转发给下一跳 P2（10.1.23.3）。虽然交换前后的标签数值相同，但它们分别属于不同设备的本地标签空间，代表不同设备分配的标签映射。

接下来，P2（AR3）收到 MPLS 报文后，同样根据最外层公网标签查询 MPLS 标签转发表。

在 P2 上执行：

<AR3> display mpls lsp

根据此前的实际输出，P2 存在以下记录：

FEC             In/Out Label    In/Out IF
10.4.4.4/32     1026/3          -/GE0/0/1

其中，出标签 3 表示 Implicit NULL（隐式空标签）。由于 P2 是该公网 LSP 的倒数第二跳，因此执行 PHP（Penultimate Hop Popping，倒数第二跳弹出） 操作，直接移除报文最外层的公网标签 1026，而不是将标签值 3 压入报文。

此时，原本的双层标签结构：

[公网标签 1026][私网标签 1028][IPv4 报文]

变为：

[私网标签 1028][IPv4 报文]

随后，P2 将只携带私网标签的 MPLS 报文通过 GE0/0/1 转发给 PE2（AR4）。

#### 5.3.3 数据由出站 PE2 转发给远端 CE2

当数据包经过 MPLS 骨干网到达出站 PE2 时，由于 P2 已经执行 PHP 操作，因此报文只剩下内层私网标签。

PE2 收到报文后，根据私网标签查找本地 MPLS VPN 标签转发信息，确定报文对应的 VPN 路由及转发方式。

在 PE2 上执行：

<AR4> display mpls lsp

根据此前的实际输出，PE2 存在以下 BGP LSP 记录：

LSP Information: BGP LSP

FEC             In/Out Label    Vrf Name
1.1.6.1/32      1028/NULL       vpna

其中，1028/NULL 表示 PE2 为 VPN 实例 vpna 中的 1.1.6.1/32 路由分配了本地私网入标签 1028，并且不需要继续使用 MPLS 出标签。

PE2 根据该私网标签匹配到相应的 VPN 转发信息，移除私网标签，将数据包还原为普通 IPv4 报文。

由于 PE2 已经配置以下 VPN 静态路由：

ip route-static vpn-instance vpna 1.1.6.1 255.255.255.255 172.16.2.2

因此，PE2 确定访问目的网络 1.1.6.1/32 的下一跳为 CE2 的接口地址 172.16.2.2，出接口为 GE0/0/1，随后将普通 IPv4 报文转发给 CE2（AR6）。

CE2 收到报文后，发现目的 IP 地址 1.1.6.1 属于本地 Loopback0 接口，因此完成业务报文的接收。

至此，Site1 到 Site2 的业务数据通过 MPLS L3VPN 完成了跨站点转发。

数据平面总结

在 MPLS L3VPN 的静态路由实验中，当 Site1 向 Site2 的 1.1.6.1/32 发送业务数据时，CE1 首先根据本地静态路由，将普通 IPv4 报文转发给 PE1（172.16.1.1）。PE1 根据入接口绑定的 VPN 实例 vpna 查询 VPN 转发表，匹配到目的网络为 1.1.6.1/32 的 IBGP 路由，获得 PE2 通过 MP-BGP 通告的私网标签，以及 BGP 下一跳 PE2 的 Loopback0 地址 10.4.4.4。由于 BGP 下一跳并非直连，PE1 需要进行公网路由和 MPLS 隧道递归解析：首先通过骨干 OSPF 路由确定实际下一跳为 P1（10.1.12.2），随后关联以 10.4.4.4/32 为 FEC 的公网 LDP LSP，获取由下游 P1 通告的公网出标签。在此前的 LSP 输出中，该公网标签为 1026，而 PE2 为目的 VPN 路由分配的私网标签为 1028。因此，在标签值保持一致的情况下，PE1 首先压入内层私网标签 1028，再压入外层公网标签 1026，将双层标签报文转发给 P1。

P1 收到报文后，根据公网入标签 1026 查找 MPLS 标签转发表，将其交换为出标签 1026，并转发给 P2。P2 收到报文后，根据标签转发表匹配到 1026/3，由于出标签 3 为隐式空标签，因此执行 PHP（倒数第二跳弹出） 操作，移除外层公网标签，仅保留私网标签 1028，随后将报文转发给 PE2。最后，PE2 根据私网标签 1028 查找对应的 VPN 转发信息，识别出目的路由 1.1.6.1/32，移除私网标签，将报文还原为普通 IPv4 报文，再根据本地 VPN 静态路由，通过 GE0/0/1 将报文转发给 CE2（172.16.2.2）。至此，Site1 向 Site2 的业务数据完成跨站点转发。

整个过程可以概括为：CE1 通过静态路由将数据交给 PE1，PE1 根据 VPN 路由及公网 LDP LSP 封装双层 MPLS 标签，P1 执行标签交换，P2 执行 PHP，PE2 根据私网标签完成 VPN 转发处理，最终通过静态路由将普通 IPv4 报文交给 CE2。

其中，公网标签负责引导业务报文沿运营商骨干网的 LDP LSP 到达出站 PE，私网标签负责出站 PE 识别对应的 VPN 转发信息，而中间的 P 路由器只需要根据公网标签完成转发，无需维护客户 VPN 路由。与 OSPF 实验相比，静态路由实验仅改变了 PE–CE 之间客户路由的获取方式，并未改变 MPLS L3VPN 的双层标签封装、标签交换及 VPN 数据转发机制。

## 6.CE-PE 之间运行 BGP 协议