---
title: 在传入连接受限的网络中接受短信平台的推送
date: 2016-12-02 23:07:14
tags: [Network, Python, Django]
---

### 背景

服务器有公网 IPv4 和 IPv6 地址。但 IPv4 地址被学校防火墙限制，不能接受教育网以外的传入连接，而 IPv6 则无任何限制，任意 IPv6 地址均可传入。所以之前在服务器上搭建的网站都是通过 CloudFlare 做 IPv6 CDN 来实现公网访问的。为了防止被q，服务器上的网站都只监听 443, nginx 的默认主页才监听 80.（顺便吐个槽，访问量低了以后域名解析好慢啊……以及第一次用 CloudFlare 没开 HTTPS 被秒 reset，真是害怕）

### 任务

我们打算做个义务维修活动的管理平台，在收到新的电脑后给同学的手机发短信确认，并提供取货号，以便在通知维修完成后取回电脑。同时要求支持同学们回复短信（回复短信的过程称之为短信上行）。可以简单地理解为“收到请回复” 233333

短信服务用的是云片网，支持上行，但下行短信的号码会变，不过相比不支持上行的阿里大于还是要好一点……它的上行短信推送机制是发 POST 到指定网址，看起来很常规（呢）

Web 组的哥们儿用 Django 肝出了活动信息管理后台。初期只支持下发短信，最近加好了接收上行短信 POST 的功能，于是现在要部署上行接受这部分功能。

### 惨痛经历

看起来接收 POST 就行了？直接把新的代码 fetch 下来，rebase 一发，按照云片网的示例 urlencode 得到测试用的信息，本地先来个 curl 测试一下。

```
ServiceObject does not exist.
```

从头走起分析。第一步是 urls.py 的路由设置，发现新加的 ^post_the_feedback/$ 规则在最后面，上面有一个 ^(?<short_link>).*/$ 的短链接识别规则。估计是先匹配了短链接规则，导致 post_the_feedback 路由到错误的 view 去了。

把 post_the_feedback 这条规则移到短链接前面去就好了。满怀希望地再来一个 curl……

```
HttpResponse required but got None.
```

这个请求是由一个函数来 handle 的，而不是一个类，看起来简单一点。这个函数确实是啥都不返回的，我猜 Django 一定要返回点 HTTP 相关的东西才能知道怎么构造回应。

然而我没写过 Python……一起调试的哥们说直接用 `return HttpResponse('Message to show')` 就好了，但是试了下这样做的 HTTP 状态码是 200, 并不能指示错误。谷歌之得到关键字参数 status 可以指定状态码。想想，卧槽 Method Not Allowed 的状态码是多少来着？继续谷歌，发现是 405, 同时发现 418 是 I'm a Teapot. 想到 StackOverflow 在用这个错误代码，于是我也卖个萌。

```
418 Unknown Error Code
```

Django 酱真是严肃呢……乖乖 405 好了。

把代码里所有的 `return` 都改成 `return HttpResponse('xxxx', status=405)`，应该好了吧？

```
HttpResponse required but got None.
```

所有的 `return` 不是都改了吗？哦，程序里只处理了 POST，GET 被忽视了 [smile] 补全这个判断以后再来

```
Invalid Signature
```

哥们说因为 APIKEY 是用的我们自己的，所以示例的签名过不去，是正常的。

那么就直接把地址提交到云片吧。咦，它自己提供了一个接口测试工具，不错，来试试。

```
（经过漫长的“测试中”以后）您的接口返回值错误，请检查
```

对照 API 文档，发现成功时应该返回 SUCCESS 或者 0, 而不是 Success。（之前补 HttpRepsonse 的时候居然猜中了这个 Success 6666 可惜大小写不对）

```
您的接口返回值错误，请检查
```

What the huck? 看了下 `supervisorctl tail ca_service stderr`，没有任何问题。nginx 的日志没开，于是加 `access_log` 打开日志，再去点测试。

```
（access 日志里是空的）
```

黑人问号？？？日志呢？我的日志呢？有哪位知道 nginx 的 access log 是带缓存需要过一会再看的吗？curl 以后日志不都是秒出的吗？

```
+1s +2s +了很多s 以后日志里依然是空的
```

卧槽这请求就没发过来吧？考虑到 CloudFlare 算是在墙外的，国内服务访问不到也算正常（虽然拿起手机发现联通 4G 访问这网页没有任何问题），于是跟旁边哥们借了 VPS 想看看国内 IP 能不能连通。这哥们域名没有备案，国内 VPS 就不给开 80 端口，于是他也只开了 SSL。

把他的 VPS 上 nginx 的 `access_log` 也打开，域名下随便打了个路径到云片网，点击测试

```
（漫长的“测试中”以后）您的接口返回值错误，请检查
```

```
（access 日志里是空的）
```

我的内心是崩溃的。我的日志呢？？？还有，你倒是告诉我怎么错误了啊？？？

然后大脑里就没有然后了。随手把域名换成 IP，点测试，秒回错误。

诶，这次速度这么快，说明它确实连上了服务器吧？

旁边哥们说，我们一直都是用的 https，会不会是证书太新（CloudFlare 需要 SNI 支持，而他的 VPS 用的是 Let's Encrypt）而导致不能用？

66666 好思路，我喜欢。于是去掉s。

```
（秒回）您的接口返回值错误，请检查
```

```
xxx.xxx.xxx.xxx [02/Dec/2016:21:40:30 +0800] "POST /post_the_feedback/ HTTP/1.0" 200 17 "-" "Jakarta Commons-HttpClient/3.1"
```

我了个大槽，有请求了！你不支持 https 倒是直接报 URL Scheme 不支持啊！

这下就有眉目了，然而未备案的 VPS 不能用 80, CloudFlare 为避免被q也不能用 80, 要怎么办呢？

算了，还是拿 VPS 的 80 试试吧。不用域名连接，直接用 IP，应该安全一点吧。那么接下来的任务就是把 VPS 的 80 转发到服务器上。（我脑子里居然一闪而过 iptables）

反正都是要用 `proxy_pass` 的，走去国外也太慢了，不如直接做个内网穿透吧。正好 ngrok 有了个叫 frp 的替代品，还没试过，可以玩玩。

在 VPS 上和服务器上下载 GitHub 上的 release（略去对穿q网速的吐槽），两边配置好。

```
# frpc.ini
[common]
...

[web_service]
type = http
local_ip = 127.0.0.1
local_port = 80
```

```
# frps.ini
[common]
vhost_http_port = 8080
...

[web_service]
type = http
custom_domains = service.ca.com
auth_token = xxxxxx
```

并且在 VPS 的 nginx 下为需要的路径配置代理

```
location /post_the_feedback
{
    proxy_pass http://127.0.0.1:8080;
}
```

然后访问 http://VPS_IP/post_the_feedback 和 http://VPS_DOMAIN/post_the_feedback 都 time out，不带路径直接访问 8080 端口也是。

卧槽，这软件搞什么搞？开个 debug 日志，输出到控制台（默认输出到文本文件，出错了看日志才知道。一声不吭地退出了，我还以为是转入后台了）

```
（访问的时候没有任何新日志产生）
```

尴尬。

继续看说明，哥们说 `custom_domains` 要改解析，我们改不了，会不会是这个原因？把 VPS 的域名加到 `custom_domains` 试试？

66666 好思路，我喜欢。加上以后 http://VPS_DOMAIN:8080 出现了 nginx 的默认欢迎画面，这是我们期望的。同时 frp 也出现了新的日志。看来，Host 字段如果不在 `custom_domains` 里面，frp 就不会响应（跟 SS 一样，对于传入请求的错误不会记录）

但是 nginx 那边的代理还是time out。一看配置文件，诶，我没写 Host？

```
location /post_the_feedback
{
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
}
```

好的，一切顺利（是吗）。用 http://VPS_DOMAIN/post_the_feedback 可以看到本地 nginx 的错误消息了。那么现在就需要把 Host 纠正到 service.ca.com 来让本地的 nginx 找到正确的配置。

frp 的文档说有一个 `host_header_rewrite` 参数可以在转发的时候改 Host。加上 `host_header_rewrite = service.ca.com` 访问 http://VPS_DOMAIN/post_the_feedback。

```
404 Not Found
```

怎么跟没加一样……难道 release 版本不够新？我也懒得下 Go 了，hack 一下吧。（省略由于两端参数位置对应 common 和 tunnel 小节不同导致的混乱，比如 `host_header_rewrite` 的文档示例中要求在 frpc 端，但 tunnel 定义里却有一个 frps 端才有的 `custom_domains` 选项）

```
location /post_the_feedback
{
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host service.ca.com;
}
```

嗯，又 time out 了。经过我已经记不清的在两端移动和修改 `custom_domains` 的尝试以后，终于不再 time out。定睛一看，

```
404 Not Found
```

加的 Host 字段被吃了吗？突然想起本地服务为防止被 q 都没有监听 80 端口 [smile] 到头来这是被谁坑了呢……

在义务维修平台的 server 块加了 80 的监听，访问 http://VPS_DOMAIN/post_the_feedback。

```
Method not supported
```

**那一刻，终于感觉世界明亮了起来。**

赶紧去云片网添加了这个地址，点测试

```
（秒回）您的接口返回值错误，请检查
```

哟，你是缓存了这个域名的 A 记录吧？这次这么快就*错误*啦？

……为什么还是错误呢？nginx 不方便记录 POST 的全部内容，那只能抓包看了。`yum install wireshark`

```
# wireshark
zsh: command not found
```

？？？

```
# tshark
Running as user "root" and group "root". This could be dangerous.
Capturing on 'nflog'
```

……将就用吧。

抓到包发现是返回了 Invalid Signature。难道签名算法不对？上次阿里大于的算法也是搞了一整晚才发现默认参数不对。

哥们说，这个测试都没说要用哪个 APIKEY 来测，而程序里的 APIKEY 是自己的，是不是有问题？

66666 我喜欢。继续看抓包，发现云片发来的数据就是它的 API 手册里那个，用的 APIKEY 是 0000. 于是把程序里的 APIKEY 也改成 0000。

```
您的接口返回值错误，请检查
```

```
（抓包）list index out of range
```

哥们说，这个测试的手机号是无效的，我们的数据库里没有，因此 `filter` 这个手机号结果是空的。所以这说明签名算法没问题，直接上生产吧。

666666 APIKEY 改回自己的，直接上生产环境。可是这不知道结果怎么办呢？虽然回复的短信会存下来，但后台还没有显示。

谷歌到 CentOS 底下 Wireshark 的图形界面叫做 `wireshark-gnome` （KDE 不服，但尝试以后表示只有 -gnome），终于顺手了。

发一条短信出去，看抓包

```
Invalid Signature
```

暴走。刚好好的怎么就炸了。与哥们一致同意把签名校验注释掉。

```
500 Internal Server Error
（设 DEBUG = True 重启）
string '2016-12-02+21:50:00' is not of format 'YYYY-MM-DD HH:MM:SS'
```

在获取时间的字符串以后加 `.replace("+"," ")` （省略手残掉空格以后再次 500）

```
500 Internal Server Error
string index out of range
```

哪里还有空的 string list 呢？点开 Local vars 一看，`recipient_list` 是空的。代码里它的内容是 `responsible_email`，是活动负责人的邮箱，但后台还不支持设置活动负责人。硬编码到公共邮箱吧。

```
500 Internal Server Error
[Error 111] Connection refused
```

我想起来前几天邮件参数全是空的然后我就删了。有这个参数要求就说一声啊……

从其他项目复制邮件配置过来以后，终于__的能用了……

（省略若干邮件内容优化）

赶着把代码都提交了，还好回宿舍没有迟到被关在门外。

好了，流水帐就到这里。最后祝您，身体健康。
