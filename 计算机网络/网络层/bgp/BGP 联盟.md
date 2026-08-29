# BGP 联盟

## 1.联盟的基本概念

在一个 AS 中，如果 iBGP 的会话数量较多，管理起来将会显得麻烦，尤其是受 iBGP 水平分割的影响，为了解决该问题，除了使用路由反射器之外，还可以使用 BGP 的联盟（Confederation）。**<font color="red">联盟的概念就是将一个 AS 划分为若干个子 AS。每个子 AS 内部建立 iBGP 全连接关系（联盟 iBGP 邻居），子 AS 之间建立 eBGP 连接关系（联盟 eBGP 邻居），但联盟外部 AS 仍认为联盟是一个 AS</font>**。

## 2.联盟的特点

配置联盟后，原 AS 号将作为每个路由器的联盟 ID。在联盟内部具有以下特点：

1. 在联盟内部将会保留联盟外部的 **`next_hop`** 属性；
2. 通告给联盟内的路由的 MED 属性在整个联盟范围内保留；
3. Local Preference 属性在整个联盟范围内保留，而不只是在通告的成员 AS 内；
4. 在联盟内将成员的 AS 号加入 **`AS_PATH`** 中，但不会将联盟内的 AS 号通告到联盟之外。在联盟中，**`AS_PATH`** 属性又添加了两种类型 **`AS-CONFED-SEQUENCE`**、**`AS-CONFED-SET`**，默认联盟将成员的 AS 号以 **`AS-CONFED-SEQUENCE`** 的形式在 **`AS_PATH`** 当中列出，如果在联盟内配置了聚合，AS 号将以 **`AS-CONFED-SET`** 形式列出；
5. **`AS_PATH`** 中的联盟 AS 号用于避免环路，但是在联盟选择最短的 **`AS_PATH`** 路径时不会比较联盟 AS 号；
6. 联盟内相关的属性传出联盟时将会被自动删除，无需过滤子 AS 号等信息操作。

我们以如下的拓扑图来对上述特性进行验证，AS 100 被分为三个子 AS（65001、65002、65003），使用 AS 100 作为联盟 ID，此时不需要采用 iBGP 的全互联，连接数从 10 条减少到了 5 条。如果没有配置联盟，AS 100 内部都是 iBGP 邻居，配置联盟以后形成了联盟的 eBGP 和联盟的 iBGP 邻居，在联盟成员 AS 内部可以使用全互联或 RR，而在联盟成员 AS 间使用联盟 **`AS_PATH`** 来避免环路。

>R1-R5 之间的连接是为了验证 **`AS_PATH`** 中的联盟 AS 号用来避免环路，以及在联盟选择最短的 **`AS_PATH`** 路径时不会比较联盟 AS 号。如果不考虑实验用途，可以不增加。

<div align="center">
    <img src="bgp_static/20.png" width="750"/>
</div>

### 2.1 验证 **`AS_CONFED_SEQUENCE`** 和联盟成员 AS 对外隐藏

从下面的结果中可以看到，R3 没有把 **`100.1.1.0/24`** 继续通告给 R5，直接原因是这条 BGP 路由的 **`NEXT_HOP`** 字段（**`10.1.12.2`**）R3 无法解析，R3 的 IP 全局路由表中没有 **`10.1.12.2`** 的路由条目，该路径被判定为无效路径，不具备参与最优路径选择的资格，因此无法成为 best/select/active 路由，也就不会作为最优 BGP 路由继续向 R5 通告。

```java{.line-numbers}
<R1>display bgp routing-table 100.1.1.0

 BGP local router ID : 1.1.1.1
 Local AS number : 65001
 Paths:   1 available, 1 best, 1 select
 BGP routing table entry information of 100.1.1.0/24:
 From: 10.1.12.2 (2.2.2.2)
 Route Duration: 01h08m13s  
 Relay IP Nexthop: 0.0.0.0
 Relay IP Out-Interface: GigabitEthernet0/0/0
 Original nexthop: 10.1.12.2
 Qos information : 0x0
 AS-path Nil, origin igp, MED 0, localpref 100, pref-val 0, valid, internal-conf
ed, best, select, active, pre 255
 Advertised to such 1 peers:
    10.1.13.3
<R3>display bgp routing-table 100.1.1.0

 BGP local router ID : 3.3.3.3
 Local AS number : 65002
 Paths:   1 available, 0 best, 0 select
 BGP routing table entry information of 100.1.1.0/24:
 From: 10.1.13.1 (1.1.1.1)
 Route Duration: 01h05m59s  
 Relay IP Nexthop: 0.0.0.0
 Relay IP Out-Interface: 
 Original nexthop: 10.1.12.2
 Qos information : 0x0
 AS-path (65001), origin igp, MED 0, localpref 100, pref-val 0, external-confed,
 pre 255
 Not advertised to any peer yet
<R5>display bgp routing-table 100.1.1.0
Info: The network does not exist.

<R3>display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 10       Routes : 10       
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        3.3.3.3/32  Direct  0    0           D   127.0.0.1       LoopBack0
      10.1.13.0/24  Direct  0    0           D   10.1.13.3       GigabitEthernet0/0/0
      10.1.13.3/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/0
      10.1.34.0/24  Direct  0    0           D   10.1.34.3       GigabitEthernet0/0/1
      10.1.34.3/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/1
      10.1.35.0/24  Direct  0    0           D   10.1.35.3       GigabitEthernet0/0/2
      10.1.35.3/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/2
```

这是因为根据 RFC 5065，It is reasonable for Member Autonomous Systems of a confederation to share a common administration and Interior Gateway Protocol (IGP) information for the entire confederation.**<font color="red">It is also reasonable for each Member-AS to run an independent IGP. In the latter case, the NEXT_HOP may need to be set using policy (i.e., by default it is unchanged)</font>**. 也就是如果各 Member-AS 使用独立的 IGP，那么原始 **`NEXT_HOP`** 可能在另一个 Member-AS 内不可达（使用的 IGP 协议互相独立，不共享路由信息），此时就需要通过路由策略修改 **`NEXT_HOP`**，或者让底层 IGP 能够到达这个地址。

所以我们在 R3 和 R5 上进行如下配置：

```java{.line-numbers}
[R3]ip route-static 10.1.12.0 24 10.1.13.1
[R5]ip route-static 10.1.12.0 24 10.1.35.3
```

配置完成之后，R3、R5、R6 上的 **`100.1.1.0`** 路由相关的信息如下所示。在 R1 上 AS-PATH 是 Nil，因为 R1、R2 都在 Member-AS 65001，同一个 Member-AS 内通过 iBGP 传播时不修改 **`AS_PATH`**。该路由传递到 R3 之后，AS-PATH 属性变为 **`(65001)`**，到了 R5 之后变为 **`(65002 65001)`**，但是到了联盟之外的 R6 之后变为 100。

```java{.line-numbers}
[R3]display bgp routing-table 100.1.1.0

 BGP local router ID : 3.3.3.3
 Local AS number : 65002
 Paths:   1 available, 1 best, 1 select
 BGP routing table entry information of 100.1.1.0/24:
 From: 10.1.13.1 (1.1.1.1)
 Route Duration: 01h44m38s  
 Relay IP Nexthop: 10.1.13.1
 Relay IP Out-Interface: GigabitEthernet0/0/0
 Original nexthop: 10.1.12.2                                      // BGP NEXT_HOP 属性本身仍然是 10.1.12.2，没有被 R1/R3 改写；静态路由只是让本地设备能够递归解析这个 NEXT_HOP，对于下面的 R5 同理
 Qos information : 0x0
 AS-path (65001), origin igp, MED 0, localpref 100, pref-val 0, valid, external-
confed, best, select, active, pre 255
 Advertised to such 3 peers:
    10.1.13.1
    10.1.35.5
    10.1.34.4
[R5]display bgp routing-table 100.1.1.0

 BGP local router ID : 5.5.5.5
 Local AS number : 65003
 Paths:   1 available, 1 best, 1 select
 BGP routing table entry information of 100.1.1.0/24:
 From: 10.1.35.3 (3.3.3.3)
 Route Duration: 00h03m24s  
 Relay IP Nexthop: 10.1.35.3
 Relay IP Out-Interface: GigabitEthernet0/0/0
 Original nexthop: 10.1.12.2
 Qos information : 0x0
 AS-path (65002 65001), origin igp, MED 0, localpref 100, pref-val 0, valid, ext
ernal-confed, best, select, active, pre 255
 Advertised to such 2 peers:
    10.1.15.1
    10.1.56.6
<R6>display bgp routing-table 100.1.1.0

 BGP local router ID : 6.6.6.6
 Local AS number : 200
 Paths:   1 available, 1 best, 1 select
 BGP routing table entry information of 100.1.1.0/24:
 From: 10.1.56.5 (5.5.5.5)
 Route Duration: 00h02m59s  
 Direct Out-interface: GigabitEthernet0/0/0
 Original nexthop: 10.1.56.5
 Qos information : 0x0
 AS-path 100, origin igp, pref-val 0, valid, external, best, select, active, pre
 255
 Not advertised to any peer yet
```

这也就验证了在联盟内将成员的 AS 号加入 **`AS_PATH`** 中，但不会将联盟内的 AS 号通告到联盟之外，并且默认联盟将成员的 AS 号以 **`AS-CONFED-SEQUENCE`** 的形式在 **`AS_PATH`** 当中列出。

根据华为文档，**`AS_CONFED_SEQUENCE`** 在 CLI 中使用圆括号 **`()`** 表示，根据 RFC 5065, 当 BGP 路由从一个 Member-AS 通告给同一 Confederation 中的另一个 Member-AS 时，发送方必须在 **`AS_PATH`** 中记录自身的 Member-AS Number，具体通过 **`AS_CONFED_SEQUENCE`** 实现。

- 如果 **`AS_PATH`** 的首个 Path Segment 已经是 **`AS_CONFED_SEQUENCE`**，发送方直接将自己的 Member-AS Number 添加到到该 Sequence 中。例如，Member-AS 65002 收到 **`(65001) 64512`** 后再通告给 Member-AS 65003，路径将变为 **`(65002 65001) 64512`**。
- 如果首个 Segment 不是 **`AS_CONFED_SEQUENCE`**，发送方则新建一个 **`AS_CONFED_SEQUENCE`**，并将自己的 Member-AS Number 放入其中，例如 **`64512 64496`** 经 Member-AS 65001 向其他 Member-AS 通告后变为 **`(65001) 64512 64496`**。
- 如果原 **`AS_PATH`** 为空，发送方同样需要创建一个新的 **`AS_CONFED_SEQUENCE`**，并写入自身的 Member-AS Number。例如 Member-AS 65001 将本地产生的路由通告给 Member-AS 65002 时，**`AS_PATH`** 将由空变为 **`(65001)`**。

### 2.2 验证外部 NEXT_HOP 在整个 Confederation 内保留

