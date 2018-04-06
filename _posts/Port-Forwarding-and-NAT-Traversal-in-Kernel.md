---
title: 通过隧道在内核中直接实现内网穿透 + 端口转发
date: 2018-04-06 19:07:35
tags: Network
---

目前常见的内网穿透大多是用 frp、ngrok 等程序实现的。但个人觉得，数据转发这种没有用户态程序交互需求的事情，应该在内核中直接完成，没必要到用户态绕一圈。这类程序中 6relayd 可能还勉强算得上个特例，只靠内核很难搞定 ND 混乱环境中 IPv6 的路由（但是我还是宁愿去魔改 ndsend 来实现内核态直接路由），但内网穿透 + 端口转发这种事是完全可能的，它所需要的各项技术都非常完善了，也没有外部限制，没有任何理由放在用户态完成。此外，用户态实现这个功能还有一个巨大的问题：位于内网的其他程序无法得知访问源 IP，ACL 或者访客统计做起来都十分尴尬。

要实现内核态内网穿透 + 端口转发，分析过程很直觉，操作嘛…… 循序渐进吧。具体如下。

（下列说明中涉及的设备有三个：LAN 主机、家里路由器、VPS，拓扑大家都懂就不画了）

## 分析：数据通路设计

需求是“内网穿透 + 端口转发”，将两部分分开来考虑。这里先不管具体怎么实现，心里对于数据怎么走有个数就行。

先考虑大家都很熟悉的端口转发。通常的端口转发就是在自家路由器上配一条 DNAT 规则，把发到路由器的 WAN 端某端口的数据包改了目标到 LAN 后再路由，就可以发到 LAN 内的主机上了。但在内网环境下，外网的数据包直接发会发不进来。我们手里还有一台有公网地址的 VPS，可以借用它的 IP. 那么：

### Goal 1: 让 VPS 把它收到的数据包转发到家里的路由器

嗯，又是一个“转发”的需求，所以又是一个 DNAT. Goal 1 完成，很轻松。

DNAT 目标显然就是家里的路由器了。家里路由器在内网，怎么让 VPS 能够穿过前置 NAT 设备把数据包发进来呢？那么：

### Goal 2: 让 VPS 能够穿越 NAT 发送数据

穿越 NAT 只有一个思路：家里路由器主动发起连接到 VPS，这样前置 NAT 设备就会记住我们这条连接，VPS 在这条连接上返回的数据就能穿越 NAT 回到家里了。（你说 UPnP？抱歉，我这里 NAT 太多层了）当然，这条连接得一直连着，不能断了。我们自己当然不会主动去断开连接，但不走数据的话前置 NAT 会认为这条连接死了，很快就会清除连接条目，然后这条连接走不到目标，就真的死了 :-( 要解决这个问题也很简单，只要不停地在这条连接上收发数据就可以。

好了，仅需考虑三秒钟，你就会设计出全新的数据通路。现在可以考虑实现细节了。

## 分析：数据通路构建

上面是两个 DNAT，那么就要知道 DNAT 目标地址。路由器到 LAN 主机的目标地址就是 LAN 主机，VPS 到路由器的目标地址是什么呢？如果 VPS 和路由器的关系能像路由器和 LAN 主机一样（在同一个子网内），那么直接设到这个子网内路由器的地址就好了。

又是同一子网，又是路由器发起连接，这很明显就是 VPN 的特征了！VPS 上建立 VPN 服务器，路由连上，形成一个虚拟局域网。然后 VPS 把所需端口 DNAT 经 VPN 隧道发到路由器，路由器再 DNAT 转发给 LAN 主机，这条通路就完成了。

想到这里，有没有觉得浪费了那么多久经考验的稳定的工具？

考虑一点特殊的环境因素，以及前面所提到的 NAT 连接保活，我们这里选择的 VPN 工具是 [censored]. 其设置过程相对简单，可以参考 [censored]. 首先 [censored]，接下来打开 [censored]，[censored]. 注意 [censored] 要多考虑一下，参考 [censored]. 最后 [censored] 之后就可以了。

## 实现：iptables 的简单玩法

这里假设 VPS 公网是 45.54.1.1，VPN 内网是 10.10.10.1；路由器 VPN 内网是 10.10.10.2，LAN 子网是 192.168.1.0/24；LAN 主机是 192.168.1.99，需要将 5000~6000 这么多端口都从 45.54.1.1 转发到 LAN 主机。

两条 DNAT 非常容易，闭着眼睛都能打出来。

```shell
(VPS) # iptables -t nat -A PREROUTING -p tcp -d 45.54.1.1 --dport 5000:6000 -j DNAT --to-destination 10.10.10.2
(Router) # (没啥命令，把 VPN 加到 WAN Zone 里，然后直接在 web 上像常规一样设置转发就可以了)
```

然后试一试…… 嗯，不通？

？？？

在各个设备的各个接口上用 tcpdump 抓包，可以看到数据包正确地流到了 LAN 主机上，但是 LAN 主机回应的数据包发到了路由器的默认出口上，而不是 VPN 接口 :-) “默认路由”是不是很厉害？

## 实现：策略路由

传入方向就像上面说的，很简单，但传出方向就遇到多 WAN 口的情况了。策略路由不可避。

但是这里有个问题，内网主机收到的数据包中的源 IP 就是原始源 IP，回应的数据包的源和目标与 LAN 发出的正常请求无法区分，数据包自身没有任何特征能够让路由器知道“这是在回应从 VPN 来的请求，所以我要把它从 VPN 发出去”。这种情况下怎么让路由器知道这个数据包要从 VPN 走呢？而且，为毛同一个连接的数据包会走两个不同的出口啊，哪来的不能回哪去啊？

……好像听到了“同一个连接”这种说法？如果能知道这个包所在连接是哪条，就能知道通过连接的传入接口知道它是不是在回应 VPN 来的包，进而就能跟 LAN 正常传出的数据包区分了，策略路由有望。

看一下 `ip rule help`，它支持的匹配条件里，源和目标 IP、源接口肯定是用不成了，说了好多遍跟正常数据包一样的。那么还剩下一个 `fwmark` 可以用。有没有觉得很眼熟？iptables 里 mark 可是铺天盖地地在用。而如果跟 iptables 联动，那么策略路由的匹配条件就可以达到 iptables match 的丰富程度。也就是说，iptables 里的 conntrack 也可以用咯。既然 conntrack 可以用了，那么“同一个连接”也就能识别咯。于是，整个链条就通了：通过 conntrack 标记出 VPN 接口传入的连接，然后让这条连接上的数据包都走 ip rule 为它选择的 VPN 出口，这个问题就解决了。

> 一点小小的问题：conntrack 是针对连接打标，ip rule 期待的是对数据包打标。这俩显然不是打在同一个对象上。那么就得想办法把连接的标记打到数据包上，这样才能根据连接来选路。好在 conntrack 给这种看起来很常见（实际上也很常见）的需求提供了方便的 target，可以说是十分贴心了。

这部分的操作正如上面所述的三步：

1. 对 VPN 来的连接打标
2. 把连接的标记打到数据包里
3. 让带有这个标记的数据包通过 VPN 出去

```shell
(Router) # iptables -t mangle -A PREROUTING -i tun0 -j CONNMARK --set-mark 0x1234
(Router) # iptables -t mangle -A PREROUTING -m connmark --mark 0x1234 -j CONNMARK --restore-mark
(Router) # ip rule add fwmark 0x1234 lookup 1234
```

0x1234 的 mark 和 1234 的路由表当然是随便起。相对于前面这些考虑，设定路由表就简单多了。

```shell
(Router) # ip rule add table 1234 default via 10.10.10.1 dev tun0
(Router) # ip rule add table 1234 192.168.1.0/24 dev br-lan
```

> 是不是觉得后面一行 `192.168.1.0/24` 的路由多余了？
>
> 要注意上面 iptables 是在 PREROUTING 打标记的，这时候还没有进行路由判决。等到了判决的时候，数据包已经有 mark 了，也已经 DNAT 了。如果没有这条路由，这个目标 192.168.1.99 的数据包进入 1234 这张表来查，就没有目标发不出去了。

然后再去试一下，哇，土制 QuickConnect！

## 实现：:-)

需求总是无止境的，我今天穿透一个 DSM，明天想在外面远程配置穿透一个 nginx 咋办？

反正就是再映射一个端口，很简单，去 VPS 上多加一条，然后从 2222 端口连回路由器自己的 ssh 再映射 nginx：

```shell
(VPS) # iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 10.10.10.2:22
```

然后开一把 ssh…… 连…… 接…… 超…… 时…… 怕不是 VPS 到家里被 q 了？VPN 被发现了？高峰期 QoS 了？

？？？

这个时候需要翻出 iptables 各链规则流经顺序的经典图（图源[维基](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)）：

![Netfilter-packet-flow](../img/Port-Forwarding-NAT-Traversal-in-Kernel/Netfilter-packet-flow.svg)

> 注意一点，本地发包的起始点不是左边的 start，而是中间顶上的 local process. 本地收包和其他所有情况都是左边 start. 仔细想想也是这样，不然本地收包都经过了 local process 了还能接着往下走？并且本地发包是从 L7 往下走的，也不是像 start 一样从 L2 往上走。

本地发包是不经过 PREROUTING 的，无法被 `--restore-mark`，因此还是走了默认路由表。但收包经过 PREROUTING，因此连接的 mark 还是有的。这里要让本地发包也按 connmark 走的话，在 mangle OUTPUT 上 `--restore-mark` 即可。此外，也可用源地址加一条策略路由条目：`ip rule from 10.10.10.2 lookup 1234`。后者是在本机做服务器（而非路由器）时实现源进源出的常见配置方法，我这里就直接用后一种了，毕竟指令比 iptables 短。

## 总结

好啦，思考三秒钟，实现两小时。不过以后再遇到这种设计，应该就很快了吧（flag）

需要的指令如下：

```sh
# 设置 VPS 到路由的转发
(VPS) # iptables -t nat -A PREROUTING -p tcp -d 45.54.1.1 --dport 5000:6000 -j DNAT --to-destination 10.10.10.2

# 设置路由到 LAN 主机的转发
(Router) # (把 VPN 接口加到 WAN Zone 里，然后直接在 web 上像常规一样设置转发)

# 设置 LAN 主机回应 VPS 穿透来的请求时的回程路由
(Router) # iptables -t mangle -A PREROUTING -i tun0 -j CONNMARK --set-mark 0x1234
(Router) # iptables -t mangle -A PREROUTING -m connmark --mark 0x1234 -j CONNMARK --restore-mark
(Router) # ip rule add fwmark 0x1234 lookup 1234

# 设置路由器自身回应 VPS 穿透来的请求时的回程路由
(Router) # ip rule from 10.10.10.2 lookup 1234

# 回程路由表
(Router) # ip rule add table 1234 default via 10.10.10.1 dev tun0
(Router) # ip rule add table 1234 192.168.1.0/24 dev br-lan
```

生活在这样一个奇妙的网络环境下，确实是可以促进学习的……

