---
title: 某奇怪的IPv6故障
date: 2016-07-29 21:13:08
tags: [IPv6, Network]
---

其实这篇日志应该是在学校的时候就写的，但是因为杂七杂八的各种原因拖到现在。不过我觉得这么奇妙的故障一定要写一下，不然这辈子都可能不会遇到第二次了……

<!--more-->

## 故障表现

起初只是因为看YouTube的时候发现IPv6的速度总是起不来，于是在校内的测试服务器上`dd`了个文件来下载，发现速度确实**只有4MB/S**。偶然瞟了一眼路由的显示屏，发现卧槽**下行速度居然已经11MB/S了**！确认了没有其他设备正在下载，就开始检查这个问题。

## 排查过程

这是我的网络拓扑图：

![Topology](/img/strange-ipv6-failure/1-Topology.png)

此外，这里的测试服务器跟我所在的/64前缀并不一样。

### Let me see...

首先来从那台测试服务器上`ping6`我的内网设备：

![Ping_from_peer](/img/strange-ipv6-failure/4-Ping-peer.png)

可以看到除了第一个包以外，所有的包都有两个回应，duplicate了。

因确斯丁。在路由上抓两个包看看：

![Packet_capture_on_router](/img/strange-ipv6-failure/3-Ping-router.png)

可以发现，在网关（厂商显示Hangzhou）对我的路由发出数据包后，一个华硕设备对我的路由发出了除了Hop Limit减了1以外完全一样的数据包。

我们知道，IPv6里的Hop Limit就相当于TTL，数值减一这说明这台华硕的设备**把它自己作为路由来转发了这个数据包**，而不是桥接过来的。

来，先把这个作死的MAC地址记着：08:60:6e:1d:f8:7d

### TCP: Wait!

每个包都重传，不光强行占用下行带宽，TCP也一再认为传输错误，造成大量重传，导致速度低得可怕。测试中甚至出现过路由下行11MB/S，curl只有60KB/S的情况……

这是一张`curl`表现还能看的图：
![curl](/img/strange-ipv6-failure/5-curl-pc.png)

这是我的局域网内设备访问HTTPS网站过程中的重复收包情况：

![Packet_capture_on_router](/img/strange-ipv6-failure/2-SSL-Router.png)

可以发现跟`ping6`时一样，都是有这个设备在重复发包。

把过滤器设置成SSL，画面太美我不敢看：

![tcp_retransmission](/img/strange-ipv6-failure/9-tcp-retrans.png)

这就是导致我看YouTube慢的原因了。

## 好像有哪里不对？

问题只会在我的内网设备与外部设备通信时出现。如果使用路由器与外部设备通信，则没有问题。这是一张从测试服务器`ping6`路由器WAN口地址的图：

![Ping_router_WAN](/img/strange-ipv6-failure/6-Ping-router-WAN.png)

没有出现DUP标记。而且直接在路由器上`curl`测试服务器的文件，速度也是正常的（没存图）。这是为什么呢……

此外，先不讨论上级交换机支不支持记忆IPv6地址跟端口的绑定关系，MAC地址跟端口的绑定总要支持吧？可是这些数据包都写着我的MAC地址了，为什么还会被这样一台华硕设备收到并转发呢？从另一个角度说，我在路由WAN口只能抓到目标地址是我自己的IPv6数据包（广播包除外），那么这台华硕设备按理说应该也是如此，可它为什么能得到目标地址并非它自己的数据包呢？莫非它与交换机进行了交易？

## 尝试解决

### Hello, it's me

只有一个MAC地址并不好玩，起码要把IP地址给扒出来吧。

可惜校园网有攻击防护，ARP扫描扫太快了像泛洪一样肯定要小黑屋的，于是每2秒发个包，把WAN口的IP段扫一遍。

```
WAN_IFACE=`nvram get wan_ifname`
for i in `seq 1 254`;do
    arping -w 1 -c 1 -I $WAN_IFACE 218.xxx.xxx.$i
    sleep 2
done 
```

扫完以后搜一下这个MAC就得到地址（路由这个版本的`arping`输出的MAC地址会把前导0去掉，比如08:60:6e变成8:60:6e），是`218.xxx.xxx.25`. （不在那个路由旁边，得不到原始输出，以后有机会来补充）

确认一下，大概这个样子：

![arping](/img/strange-ipv6-failure/8-arping.png)

然后拿`nmap`扫一下端口，记得限定速率……没关系，吃个饭回来就好了呢 :-)

`nmap -vv --max-rate 0.5 --max-retries 1 218.xxx.xxx.25`

```
PORT     STATE    SERVICE
135/tcp  open     msrpc
139/tcp  open     netbios-ssn
445/tcp  open     microsoft-ds
5357/tcp open     wsdapi
```

度娘了一下，发现445端口蛮好玩：

`nmap -Pn --script smb-os-discovery.nse -p445 218.xxx.xxx.25`

```
Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Ultimate 7601 Service Pack 1 (Windows 7 Ultimate 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: PC201306291745
|   NetBIOS computer name: PC201306291745
|   Workgroup: WORKGROUP
|_  System time: 2016-05-08T11:44:42+08:00
```

看名字感觉是个传统的Ghost出来的系统？

`nmap -Pn --script smb-security-mode.nse -p445 218.xxx.xxx.25`

````
Host script results:
| smb-security-mode: 
|   Account that was used for smb scripts: guest
|   User-level authentication
|   SMB Security: Challenge/response passwords supported
|_  Message signing disabled (dangerous, but default)
```

好像没什么意义……连接需要用户名密码。

`nbtscan 218.xxx.xxx.25`

```
Doing NBT name scan for addresses from 218.xxx.xxx.25
IP address       NetBIOS Name     Server    User             MAC address
------------------------------------------------------------------------------
218.xxx.xxx.25   PC201306291745   <server>  <unknown>        08:60:6e:1d:f8:7d
```

没啥特殊的。

然后就没有然后了，既不会黑，也不敢黑……毕竟自己的账号还要用好久的。

我把上面这些内容（`nmap`和`nbtscan`除外）连带原始抓包都发给了学校网络中心，第二天他们回复我说“已经通知相关人员，请注意检查。以后如有问题请及时联系”。试了下`ping6`不再DUP了，感觉好了，于是很高兴地回复说“谢谢老师。我能否知道具体是什么故障呢？”，然而不再有回信了。

算了，既然好了，就不要纠结了吧。

## 只剩我自己了？

然而过了几天，YouTube又&trade;看不动了！`ping6`一看又DUP了。呵呵。

认识一个学弟，跟网络中心老师关系比较好，他自己也对这种奇妙的故障很有兴趣，也跟老师讨论过。于是我跟他打听上一次老师到底做了什么。

>我：老师上次到底做了什么啊，能管用那么几天。
学弟：老师说他们什么都没做，然而你这里自己就好了！
我：网中套路深，我想回农村……
学弟：他们还说是你的路由器有问题。
我：包都抓了还&trade;是我路由器的问题？你倒是在交换机正常、我的路由器除了NS不发任何包的情况下给我弄个从外ping内会在WAN口收到两次包的例子啊？

算了x2，网中坑爹，还是要靠自己……（内心OS：好歹这学校也不小了，怎么网中这种技术部门还踢皮球？）

为了证明这个故障与那台华硕S(he)B(ei)有关，我打算在路由下面接个pcDuino，24小时开机，不停地`ping6`外网，收集结果。同时在路由上不停地`arping`这个.25的IP以确定目标设备上线（学校虽然是DHCP，但IP很长一段时间都不会变，所以可以使用同一个IP地址来`arping`）。

家里并没有这两个脚本，都放在寝室的设备里了。

内网客户端执行结果大概是这样的：

```
===============
2016年 6月14日 星期二 10时04分30秒 CST
PING6(56=40+8+8 bytes) 2001:xxxx:xxxx:xxxx:e6f4:2333:6666:3397 --> 2001:470:20::2
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=50 time=166.054 ms
16 bytes from 2001:470:20::2, icmp_seq=1 hlim=50 time=166.346 ms

--- 2001:470:20::2 ping6 statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/std-dev = 166.054/166.200/166.346/0.146 ms
===============
2016年 6月14日 星期二 10时05分31秒 CST
PING6(56=40+8+8 bytes) 2001:xxxx:xxxx:xxxx:e6f4:2333:6666:3397 --> 2001:470:20::2
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=50 time=166.559 ms
16 bytes from 2001:470:20::2, icmp_seq=1 hlim=50 time=166.358 ms

--- 2001:470:20::2 ping6 statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/std-dev = 166.358/166.459/166.559/0.100 ms
===============
2016年 6月14日 星期二 10时06分33秒 CST
PING6(56=40+8+8 bytes) 2001:xxxx:xxxx:xxxx:e6f4:2333:6666:3397 --> 2001:470:20::2
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=50 time=166.345 ms
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=49 time=166.499 ms(DUP!)
16 bytes from 2001:470:20::2, icmp_seq=1 hlim=50 time=168.246 ms

--- 2001:470:20::2 ping6 statistics ---
2 packets transmitted, 2 packets received, +1 duplicates, 0.0% packet loss
round-trip min/avg/max/std-dev = 166.345/167.030/168.246/0.862 ms
===============
2016年 6月14日 星期二 10时07分34秒 CST
PING6(56=40+8+8 bytes) 2001:xxxx:xxxx:xxxx:e6f4:2333:6666:3397 --> 2001:470:20::2
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=50 time=166.368 ms
16 bytes from 2001:470:20::2, icmp_seq=0 hlim=49 time=167.095 ms(DUP!)
16 bytes from 2001:470:20::2, icmp_seq=1 hlim=50 time=166.390 ms

--- 2001:470:20::2 ping6 statistics ---
2 packets transmitted, 2 packets received, +1 duplicates, 0.0% packet loss
round-trip min/avg/max/std-dev = 166.368/166.618/167.095/0.338 ms
```

可以发现在10时5分33秒到10时6分33秒之间，IPv6开始变得不正常。

路由上的结果也没有取回本地存储，但判断方法是类似的。可以发现，在内网客户端变的不正常的3分钟之前，`arping`开始能够得到回应。**每一次故障都能在两侧得到对应的结果。**等回学校了我就把路由端`arping`的日志贴出来。

那么，我就放心地认为是这个设备在影响我的网络了，并且这个设备可能与交换机进行了交易。

原来网中第一次回信之后我这里没问题了，只是因为那个人没有开机……

## 所以……

我跟学弟约好放假去找网络中心老师谈谈，结果他比我放假早先回家了（？？？）我跟网络中心里的老师也不熟，总不能跟前台肛吧。于是就一直没有反映上去。

BTW，试过在路由上直接屏蔽掉这个MAC地址，然并卵，TCP重传是没有了，但它还是强行发包，占掉我的下行带宽。校园网没有带宽限制，100Mbps可以跑满，所以它这样强行发包占带宽的话我就没有余量了，速度直接砍一半，没有任何办法……

不过日志里可以发现这个人的生活规律诶（doge脸）毕竟是个早上十点以后才开机、下午有课上课没课开机、晚上玩到断网的主，想必是个死宅。~~要是让我知道了这个设备的主人是谁，能肛绝对肛死他~~
