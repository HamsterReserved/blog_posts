---
title: K3 运行 LEDE 时 BCM4366C0 的固件性能测试
date: 2017-04-02 16:49:28
tags:
---

K3 里面是 BCM4366C0，这个型号的 firmware 没有公开可用的。公开版本只到 BCM4366B1. （这样一说，会不会 BCM4366C0 就是专门用在路由器上的型号呢？正好就不提供固件，不给第三方）

算了，不扯这个，网卡本身还是受 brcmfmac 支持的，固件可以从官方的 dhd.ko 里面提取，合起来就可以用了。提取方法如下：

1. `readelf -a dhd.ko | grep 4366c0` 会看到有一个 `dlarray_4366c0` 的条目。前面有一个十六进制表示的大小，数量级大概在 0xf0000 左右，记下它，它是**固件大小**。
2. 用十六进制编辑器打开 dhd.ko，搜索 `00 F2 3E B8` 这个串，会看到有多个条目。在每个匹配处前面**一般**会有文本描述，挑一个写着 4366c0 的位置，记住它的**偏移量**. 有时候前面没有文本，就都提取出来好了。而且这个串可能在几十个字节内重复出现两次，应该取前一次作为固件起点偏移量。
3. `dd if=dhd.ko of=4366c0-fw.bin ibs=1 skip=偏移量 count=固件大小` 提取出的固件文件应该以 `00 F2 3E B8` 开头，以一段可读文本加四字节不知道啥玩意结尾。

现在可以从 Linksys EA9500、Asus RT-AC88U/AC3100/AC5300、Netgear R8500 以及 K3 自己的系统镜像中提取固件，下面将逐一测试这些固件的性能。测试将包含以下项目：

* 系统兼容性（hostapd 带起来的过程中是否有错误，是否能输出终端连接和断开信息）
* 放在特定位置的手机和电脑对 2.4GHz 和 5GHz 信号的接收强度
* 电脑测速（黑苹果 BCM4352HMB，iperf3，十线程，无线端的电脑与上面那台不是同一台，会比上面所述的电脑更近）

固件来源具体是：

* Linksys EA9500, 1.1.7.179240 Hardware Revision 1.1
* Asus RT-AC88U / AC3100 / AC5300, Koolshare Mods X7.2 （这三个路由的固件提取出来都是一样的）
* Asus RT-AC88U, 3.0.0.4-380-7266-g6439257
* Netgear R8500, Koolshare Mods (based on stock firmware) 0.2
* Netgear R8500, 1.0.2.94_1.0.79
* K3, stock firmware MTD dump, 21.4.33.212

上官网太慢没办法 [摊手]

设备放置：路由左边 1m 是（测量信号强度的）电脑，电脑左边 10cm 是手机。问我为什么不隔开一点？因为网线只有那么长。无线都设置 WPA2-PSK-AES 加密，密码 12345678.

每个固件测试完后，都通过路由器硬件开关把路由关掉重启，以避免上次上传的固件的影响。

作为基础的 LEDE 固件通过 tftp 启动，并不刷入 flash.

## Linksys EA9500 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Feb 14 2017 16:45:30 version 10.10.122.301 (r658909) FWID 01-1ebebd8a
```

```
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -40dBm

电脑 RSSI -29dBm

### 5 GHz

手机 RSSI -49dBm

电脑 RSSI -30dBm

LAN -> 5G 650Mbps

5G -> LAN 500Mbps

## Asus RT-AC88U / AC3100 / AC5300 X7.2 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: May 12 2016 11:33:40 version 10.10.69.6904 (r635567) FWID 01-a1b24c27
```

```
Configuration file: /tmp/run/hostapd-phy0.conf
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -36dBm

电脑 RSSI -27dBm

### 5 GHz

手机 RSSI -49dBm

电脑 RSSI -31dBm

LAN -> 5G 640Mbps

5G -> LAN 530Mbps

无线客户端移动的时候速度有较大变化，5G -> LAN 刚开始时偶有断流，但不是每次都断。

## Asus RT-AC88U 3.0.0.4-380-7266 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Sep 12 2016 13:26:44 version 10.10.69.6908 (r658761) FWID 01-fed440e1
```
```
Configuration file: /tmp/run/hostapd-phy0.conf
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -35dBm

电脑 RSSI -29dBm

### 5 GHz

手机 RSSI -52dBm

电脑 RSSI -30dBm

LAN -> 5G 430Mbps 稳定（改变摆位后 500Mbps 稳定）

5G -> LAN 550Mbps（客户端位置不动，速度就会在 450Mbps ~ 650Mbps 跳动）

## Netgear R8500 Stock Mods 0.2 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Feb  3 2016 23:11:36 version 10.10.69.30014 (r617090) FWID 01-8ffebd3
```

```
Configuration file: /tmp/run/hostapd-phy0.conf
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -37dBm

电脑 RSSI -24dBm（会从 -37 一路涨上去）

### 5 GHz

手机 RSSI -49dBm

电脑 RSSI -28dBm

LAN -> 5G 300Mbps（跟电脑摆放位置有很大关系，最大 500Mbps）（降低功率正常摆位可到 330Mbps）

5G -> LAN 450Mbps（跟电脑摆放位置有很大关系，最大 650Mbps）（降低功率正常摆位可到 500Mbps）

## Netgear R8500 1.0.2.94_1.0.79 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Jun 28 2016 03:21:43 version 10.10.69.30019 (r641352) FWID 01-83812d37
```

```
Configuration file: /tmp/run/hostapd-phy0.conf
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -35dBm

电脑 RSSI -25dBm（从 -30 涨上去）

### 5 GHz

手机 RSSI -52dBm

电脑 RSSI -29dBm

LAN -> 5G 500Mbps

5G -> LAN 660Mbps

## 某未知型号路由的固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Jan 25 2017 20:24:34 version 10.10.122.308 (r) FWID 01-5df4385d
```

```
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Failed to create interface mon.wlan1: -95 (Not supported)
wlan1: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan1: interface state COUNTRY_UPDATE->HT_SCAN
wlan1: Could not connect to kernel driver
Using interface wlan1 with hwaddr d8:c8:e9:fe:89:a2 and ssid "LEDE-5G"
wlan1: interface state HT_SCAN->ENABLED
wlan1: AP-ENABLED
```

### 2.4 GHz

手机 RSSI -35dBm

电脑 RSSI -30dBm

### 5 GHz

手机 RSSI -49dBm

电脑 RSSI -31dBm（从 -37 涨上来）

LAN -> 5G 440Mbps（不稳定，改变摆位后550Mbps稳定）

5G -> LAN 600Mbps（不稳定，改变摆位后650Mbps稳定）

## K3 固件

```
brcmfmac: brcmf_c_preinit_dcmds: Firmware version = wl0: Aug 19 2016 15:22:35 version 10.10.69.74 (r629731 WLTEST) FWID 01-5c0166fa
```

```
ifconfig: SIOCGIFFLAGS: No such device
Configuration file: /tmp/run/hostapd-phy0.conf
Failed to create interface mon.wlan0: -95 (Not supported)
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
wlan0: Could not connect to kernel driver
Using interface wlan0 with hwaddr d8:c8:e9:fe:89:a1 and ssid "LEDE"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED
Configuration file: /tmp/run/hostapd-phy1.conf
Could not open configuration file '/tmp/run/hostapd-phy1.conf' for reading.
Failed to set up interface with /tmp/run/hostapd-phy1.conf
Failed to initialize interface
```

（无法在同样的时间内初始化两张卡）

只有 2.4 GHz 能够建立 AP，而且建立后无法连接，要么直接不能关联，要么密码错误。

## 结论

* K3 对 BCM94352HMB（电脑下载）最大 530Mbps左右，BCM94352HMB 对 K3 最大 650Mbps 左右。这跟以前对其他路由的测试是相反的。
* 速率与客户端摆位的关系**很大**，即使是转一个角度，速率和稳定性就会有极大的改变。
* 最终确定 EA9500 最新版本的固件比较适合于这个路由器。
* R8500 官改固件里的 BCM4366C0 固件开挂了吗！？