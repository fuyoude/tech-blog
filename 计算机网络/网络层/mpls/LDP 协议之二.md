# LDP 协议之二

## 1.LDP 会话建立过程

LDP 会话是建立在 TCP 之上的，所以建立 TCP 连接之前必须知道邻居的 IP 地址，根据获知邻居 IP 地址的方式将 LDP 会话分成两类。

- 本地 LDP 会话：建立会话的两个 LSR 之间是直连的。
- 远端 LDP 会话：建立会话的两个 LSR 之间可以是直连的，也可以是非直连的。

### 1.1 邻居发现

在启动 LDP 协议后，路由器发出 Hello 报文，相邻路由器之间会通过交换 Hello 报文来完成相互问候，通过 Hello 报文可以获知邻居的基本信息，比如用来建立会话的地址（传输地址）等。然后，**<font color="red">由地址大的一方发起 TCP 连接并最终建立邻居会话，这种方式的 LDP 会话就叫本地会话</font>**。如果以手工的方式指定邻居地址，然后建立的会话称为远端的 LDP 会话。如下图所示，两直连邻居之间相互发现的过程。

<div align="center">
    <img src="mpls_static/24.png" width="450"/>
</div>

>LDP 协议的组播地址为 **`224.0.0.2`**，OSPF 的组播地址为 **`224.0.0.5`**，RIPv2 的组播地址为 **`224.0.0.9`**。

如果邻居是非直连的设备，Hello 会以单播的方式来交换，用于完成远端 LDP 会话的建立，如下图所示。

<div align="center">
    <img src="mpls_static/25.png" width="450"/>
</div>

### 1.2 会话建立

LDP 邻居之间相互发现或直接指定了邻居地址后，开始建立 TCP 连接，**<font color="red">建立完 TCP 连接后，相互发送初始化报文进行参数协商</font>**，如下所示。

<div align="center">
    <img src="mpls_static/26.png" width="450"/>
</div>

**<font color="red">如果接受对方的参数，则会回复 KeepAlive 报文，完成 LDP 会话的建立</font>**；如果期间发生错误，则会发送 Notification 报文来报告相应的错误，最终导致连接被关闭。可以通过以下命令检查 LDP 会话的建立情况。

```java{.line-numbers}
<AR4>display mpls ldp session 

 LDP Session(s) in Public Network
 Codes: LAM(Label Advertisement Mode), SsnAge Unit(DDDD:HH:MM)
 A '*' before a session means the session is being deleted.
 ------------------------------------------------------------------------------
 PeerID             Status      LAM  SsnRole  SsnAge      KASent/Rcv
 ------------------------------------------------------------------------------
 10.1.2.2:0         Operational DU   Active   0000:05:30  1322/1322
 10.1.5.5:0         Operational DU   Passive  0000:05:30  1321/1321
 ------------------------------------------------------------------------------
 TOTAL: 2 session(s) Found.
```

状态为 Operational 表示 LDP 会话已经建立成功，邻居之间可以交换标签通告等报文了。在会话建立后，双方会定期发送 Hello 报文以检测邻居状态，也会定期发送 KeepAlive 报文来检测 TCP 连接状态。

### 1.3 LDP 状态会话机

LDP 协议从邻居发现到会话建立过程历经了 4 个状态，状态之间的切换如下图所示。

<div align="center">
    <img src="mpls_static/27.png" width="450"/>
</div>

Non Existent 状态：LDP 协议的初始状态，类似 BGP 的 IDLE 状态。开始时双方用组播地址 **`224.0.0.2`** 发送 Hello 报文，并且选举主动方和被动方（**<font color="red">LSR IP 地址高的被选为主动方</font>**）。双方收到对方的 Hello 报文后，则开始建立 TCP 连接，接着是建立 TCP 连接三次握手的过程，当 TCP 会话建立成功后，双方进入到 Initialized 状态。

Initialized 状态：该状态下主动方和被动方的工作有点不同。主动方先发送 Initialization 报文，随后便转入 Opensent 状态，并等待被动方回应 Initialization 报文。被动方在这个状态下会等待主动方发送 Initialization 报文过来，**如果收到的 Initialization 报文中的参数都能接受，会给主动方回应 Initialization 和 KeepAlive 报文**，随后便转入 Openrec 状态。主动方和被动方在该状态下收到非 Initialization 报文或等待超时都会跳转至 Non Existent 状态。

Opensent 状态：这个状态是主动方发出 initialization 报文后的状态，该状态下还会等待邻居的 initialization 和 KeepAlive 报文。如果收到的 initialization 报文中的参数都能接受，则会向邻居回应 KeepAlive 报文，并且状态跳变至 Openrec 状态；如果参数不能接受或等待时间超时，则会断开 TCP 会话连接而进入到 Non Existent 状态。

Openrec 状态：不管是主动方还是被动方，**<font color="red">在收到参数正确的 initialization 报文后都会转入此状态，并且在该状态下已经向对方回应了 KeepAlive 报文并等待着邻居回应 KeepAlive 报文</font>**。如果从邻居那边按时接收到了 KeepAlive 报文，则转入 Operational 状态；如果收到其他报文或等待时间超时，则转入 Non Existent 状态。

Operational 状态：到达此状态就意味着 LDP 会话已经建立成功，可以发送和交换所有其他的 LDP 报文了。在该状态下双方通过周期性 KeepAlive 报文交换来维护邻居会话，如果 KeepAlive 超时，则立刻断开 LDP 会话并进入 Non Existent 状态；当收到错误通知报文时，也会断开 LDP 会话并转入 Non Existent 状态。

## 2.LDP 环路检测

在三层网络中使用的每种动态路由技术都有各自的防环机制，LDP 协议也有防环机制，但华为 VRP 系统默认不开启 LDP 防环功能，这是因为三层路由技术已经避免了路由环路，而 LDP 协议又是基于路由来分配标签和建立 LSP 的，所以这时报文通过 LSP 进行转发是不会出现环路的。**除非因为路由协议收敛出现问题，出现了短暂环路。标签中也有 TTL 字段，这样也避免了数据包在 MPLS 网络中无限循环**。LDP 协议本身的防环方式有以下几种。

### 2.1 规定最大跳数

如同 RIP 协议规定路由的最大跳数为 16 跳一样，LDP 协议也可以限制标签报文经过的 LSR 数量，这个特性通过在报文中使用一个叫作 Hop Count 的 TLV 来实现。LDP 报文（比如标签请求报文）每经过一个 LSR，该 LSR 都在跳数 TLV 中增加一跳，当跳数达到最大值后，环路就被检测到了，这时就终止 LSP 的建立。华为 VRP 系统默认定义的最大跳数为 32 跳。

### 2.2 TTL 处理

MPLS 报文中每经过一台 LSR，标签中的 TTL 减 1，当 TTL 减少到 0 的时候，报文最终被丢弃，通过这种方法达到防止数据包的无限循环问题。

### 2.3 路径矢量法

路径矢量法是指在 LDP 报文传递过程中，记录沿途经过的所有 LSR，就像 BGP 协议的 **`AS-PATH`** 属性一样，当一台 LSR 收到 LDP 报文后，就会将自己的 **`LSR-ID`** 添加到报文中；如果收到的 LDP 报文已经有了本地的 **`LSR-ID`**，则认为出现了环路，终止建立 LSP。