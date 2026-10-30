# LDP 协议之一

## 1.LDP 前置基础

### 1.1 MPLS 相关表项

LIB（Label Information Base）属于 MPLS 的控制平面，主要负责保存和管理 FEC 与 MPLS 标签之间的绑定关系。这些信息可以通过 LDP 等标签分发协议获得，包括目的 FEC、本地标签、下游节点发布的标签以及发布该标签的对等体等。**<font color="red">LIB 为建立标签转发表提供依据，但不直接负责报文转发，其中保存的标签信息也不一定全部用于实际转发</font>**。在 Liberal Retention 模式下，设备还可以保留来自非当前下一跳邻居的标签绑定，以便在后续路由发生变化时使用。

LFIB（Label Forwarding Information Base）**<font color="red">属于 MPLS 的数据平面，主要负责指导 MPLS 报文的实际转发</font>**。LFIB 根据有效的标签绑定及路由信息建立转发关系，包含入标签、出标签、下一跳和出接口等信息。当路由器收到 MPLS 报文时，可以根据 LFIB 中的转发规则执行标签交换（Swap）或弹出（Pop）等操作。因此，LIB 负责管理标签信息，而 LFIB 负责实际标签转发。

ILM（Incoming Label Map）**<font color="red">属于 MPLS 的数据平面，主要负责建立入标签与 NHLFE 之间的映射关系，用于处理已经携带 MPLS 标签的报文</font>**。当路由器收到 MPLS 报文时，会根据栈顶入标签查询 ILM，找到对应的 NHLFE，再按照其中的规则转发报文。例如，收到标签为 100 的报文后，通过 ILM 找到 NHLFE，执行标签交换，将标签 100 替换成 300。ILM 在 Transit 节点的作用是将标签和 NHLFE 绑定。

NHLFE（Next Hop Label Forwarding Entry）**<font color="red">属于 MPLS 的数据平面，主要负责规定报文的标签操作和下一跳转发方式</font>**。NHLFE 包含出标签、下一跳、出接口、标签操作、tunnel-id 类型等信息，可以指导设备执行 Push、Swap、Pop 等操作。

NHLFE 可以通过 ILM 或 FTN 找到，因此 ILM 和 FTN 负责查找对应的转发规则，而 NHLFE 负责确定具体的标签处理及转发行为。FEC 与 NHLFE 的映射称为 FTN（FEC-to-NHLFE），通过查看 FIB 表中 Token 值不为 0x0 的表项，能够获得 FTN 的详细信息。**FTN 只在 Ingress 存在**。

根据 RFC 3031 中对 FEC-to-NHLFE 的描述，The "FEC-to-NHLFE" (FTN) maps each FEC to a set of NHLFEs. It is used when forwarding packets that arrive unlabeled, but which are to be labeled before being forwarded. 也就是当 LSR 收到一个尚未携带 MPLS 标签、但需要通过 MPLS LSP 转发的普通 IP 报文时，典型情况下由 Ingress 节点根据目的 IP 地址确定 FEC，再通过 FTN 找到相应的 NHLFE，并执行 Push 操作，将普通 IP 报文封装为 MPLS 报文后发送给下一跳。

根据 RFC 3031 中对 ILM 的描述，The "Incoming Label Map" (ILM) maps each incoming label to a set of NHLFEs.  It is used when forwarding packets that arrive as labeled packets. 当 LSR 收到一个已经携带 MPLS 标签的报文时，典型情况下由 Transit 节点根据报文栈顶的入标签查询 ILM，找到对应的 NHLFE，再根据其中规定的操作执行 Swap、Pop 或其他标签栈处理，并将报文转发给指定的下一跳。

### 1.2 华为 MPLS 转发表的实现（动态 LDP）

报文是如何通过LSP转发的？
以支持PHP特性的LSP为例，MPLS报文在该LSP中的基本转发过程描述如下：

<div align="center">
    <img src="mpls_static/8.png" width="800"/>
</div>

入节点Ingress收到目的地址为192.168.1.1/24的IP报文，首先通过FIB表检查该目的IP地址对应的Tunnel ID值是否为0x0。若是，按IP转发流程处理；若不是，则进入MPLS转发流程，依据此Tunnel ID找到对应NHLFE中指示的出接口、下一跳、出标签（假设为Y）和标签操作类型等信息，压入标签Y（Push），封装为MPLS报文，并按NHLFE指示的出接口和下一跳继续转发。

如下所示，先查 fib，发现目的地址 10.1.44.0/24 的 FEC 对应的 tunnel-id 地址为 0x1，然后执行 display tunnel-info tunnel-id 0x1，查询到该隧道的出标签为 100，出接口为 GE0/0/1，下一跳为 10.1.12.2。这就是上面所说的，先查询 fib 表，然后发现其对应的 Tunnel-id 不为 0，则依据此 0x1 去 NHLFE 中找到相关的信息，封装成 MPLS 报文。

```java{.line-numbers}
[AR1]display fib 10.1.44.10
  Route Entry Count: 1
Destination/Mask   Nexthop         Flag  TimeStamp     Interface      TunnelID
10.1.44.0/24       10.1.12.2       DGU   t[51]         GE0/0/1        0x1
[AR1]display tunnel-info tunnel-id 0x1
Tunnel ID:                    0x1
Tunnel Token:                 1
Type:                         lsp
Destination:                  10.1.44.0
Out Slot:                     0
Instance ID:                  0
Out Interface:                GigabitEthernet0/0/1
Out Label:                    100
Next Hop:                     10.1.12.2
Lsp Index:                    1
```

ILM（Incoming Label Map，入标签映射） 用于将 MPLS 报文的入标签映射到相应的 NHLFE；NHLFE（Next Hop Label Forwarding Entry，下一跳标签转发表项） 用于指定出标签、下一跳、出接口以及标签操作类型（Push、Swap、Pop）；LFIB（Label Forwarding Information Base，标签转发信息库） 则用于保存和组织 MPLS 数据平面的标签转发信息，可以将 ILM 与 NHLFE 的关联关系作为转发表项表示。

根据 RFC 3031，Transit 节点收到带标签的 MPLS 报文后，通过 ILM → NHLFE 确定转发行为；Ingress 节点收到普通 IP 报文后，则通过 FTN → NHLFE 将报文引入 MPLS 网络。

因此，ILM 负责匹配入标签，NHLFE 负责具体的标签处理与下一跳选择，LFIB 则是承载这些转发信息的整体转发表概念，三者不是必须依次查询的三张独立表。

Transit收到该MPLS报文，根据标签Y查询ILM表中对应的Tunnel ID，并根据此Tunnel ID在NHLFE中找到进行下一跳转发所需要的出接口、下一跳、出标签、标签操作类型（标签交换，Swap），将标签Y换成标签X后继续转发。

在 AR2 上收到标签 100 的 MPLS 数据包时，根据标签 100 查询 ILM 中对应的 tunnel-id/token（也就是 0x1），然后根据这个 0x1 的 token 在 NHLFE 中找到下一跳转发所需要的出接口、下一跳、出标签等。在下面的 LFIB 中，in-label 和 in-interface 可以看成是 ILM，而 nexthop、out-label、out-interface、lsrtype 可以看成是 NHLFE。

```java{.line-numbers}
[AR2]display mpls lsp in-label 100 verbose 
                 LSP Information: STATIC LSP

  No                  :  1
  Name                :  FWD_S1
  Fec                 :  -------/--
  Nexthop             :  10.1.24.4
  In-Label            :  100
  Out-Label           :  300
  In-Interface        :  GigabitEthernet0/0/0
  Out-Interface       :  GigabitEthernet0/0/2
  LspIndex            :  0
  Token               :  0x1
  LsrType             :  Transit
  Mpls-Mtu            :  1500
  TimeStamp           :  2459sec
```

倒数第二跳的Transit收到该MPLS报文，继续上述查询动作，此时由于为Egress分给它的标签值为3（特殊标签，参见上文“MPLS标签”），所以进行PHP操作，弹出标签X（Pop）并继续将IP报文转发给Egress。

Egress节点收到该IP报文，直接根据对应的IP路由表将其转发到目的地192.168.1.1/24。

在 AR4 上，可以看到，由于配置的是静态 LSP，所以不存在 PHP 机制，因此 AR4 收到 MPLS 标签包之后，直接弹出标签，变成 IP 数据包，然后按IP转发流程处理。可以看到，在 FIB 表中，FEC 10.1.44.0/24 对应的 tunnel-id 为 0x0，因此弹出标签之后直接按照 IP 转发流程处理。

```java{.line-numbers}
[AR4]display mpls lsp in-label 300 verbose 
                 LSP Information: STATIC LSP

  No                  :  1
  Name                :  FWD_MERGE
  Fec                 :  -------/--
  Nexthop             :  -------
  In-Label            :  300
  Out-Label           :  NULL
  In-Interface        :  GigabitEthernet0/0/0
  Out-Interface       :  ----------
  LspIndex            :  0
  Token               :  0x0
  LsrType             :  Egress
  Mpls-Mtu            :  ------
  TimeStamp           :  4348sec
[AR4]display fib 10.1.44.0
  Route Entry Count: 1
Destination/Mask   Nexthop         Flag  TimeStamp     Interface      TunnelID
10.1.44.0/24       10.1.44.1       U     t[6]          GE0/0/1        0x0
```

**华为官方文档明确说明：当 Ingress 根据目的 IP 地址查询 FIB，发现 TunnelID = 0x0 时，采用普通 IP 转发；如果存在关联的非零 MPLS Tunnel ID，则可以进一步通过相应的 MPLS 转发信息完成标签操作。Transit 节点则通过入标签查找 ILM 和 NHLFE，而不是依赖目的 IP 对应的普通 FIB TunnelID。**

DU：Downstream Unsolicited（下游自主分发）

DU 模式的核心是：下游 LSR 不需要收到上游的 Label Request，就可以主动发送 Label Mapping 消息。

在 LDP 邻居关系建立后，下游设备可以为 FEC 建立标签绑定，并将该绑定主动通告给上游。上游收到标签映射后，根据自己的路由信息判断应该使用哪一个 LDP 邻居提供的标签。

需要注意，DU 并不意味着下游设备不考虑任何条件就分配所有标签。它能否向上游发布某个 FEC 的标签，还受到标签分配控制模式（Independent 或 Ordered）、标签发布策略等因素的约束

DoD：Downstream on Demand（下游按需分发）

DoD 模式的核心是：上游 LSR 需要标签时，先向下游发送 Label Request；下游收到请求后，才向上游发布对应的 Label Mapping。

DoD 并不代表每次转发 IP 报文都要重新请求标签。Label Request 是 LDP 控制平面建立标签绑定关系时使用的协议消息，已经建立的 LSP 可以持续转发大量数据报文。

它强调的是按照请求发布所需的标签映射，而非主动向邻居通告大量 FEC 的标签映射。

标签发布方式：DU / DoD，是否需要上游先请求
标签分配控制方式：Independent / Ordered，是否需要先获得下游标签映射才能向上游发布
标签保持方式：Liberal / Conservative，是否保留非当前下一跳邻居的标签映射

例如，DU + Ordered 表示：LSR 无须收到上游的请求，但通常要先获得下游的标签映射（或者自身就是该 FEC 的 Egress），才能向上游主动发布标签。

DoD + Independent 则表示：LSR 必须先收到上游请求，但可以不等待下一跳的标签映射，就为该 FEC 分配并发布本地标签。RFC 5036 第 2.6.1 节明确区分了这两种控制模式。