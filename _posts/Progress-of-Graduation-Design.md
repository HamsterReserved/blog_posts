---
title: 毕业设计记录
date: 2016-12-27 10:13:48
tags: [Frontend, Diary]
---

### 2016/12/27

今天是第一天正式开始做，目前分配的目标是平台搭建。// 考研终于结束了……

我做的是微信客户端（其实我猜应该是公众号吧，不然没有小程序的权限啊），那么任务可明确为“移动设备 HTML5 应用开发”，初期的平台搭建任务实际上是 Web 服务端搭建。按理说微信公众号的申请也算是平台搭建的任务，但我已经有自己的微信公众号了，所以就不在这里说明公众号的申请的细节了（都快要三年过去了呢）。此外，我也有自己的 VPS，因此服务器购买也不算在这里。

要做的事情就是搭建 Web 服务器了。

关于服务器选择问题，由于之前有搭建 nginx 的经历，此处就直接采用 nginx 了。

而关于动态语言的选择问题，我只会一点 PHP……虽然以前也在一些 Python / Django 的项目里挑过 bug，不过还是不太敢自己写新的项目用（不过 Python 用起来确实比较舒服，起码不容易小指骨折）

具体过程如下：

1. 登录到 VPS 上，看看源里有没有现成的程序可以用

 源里是有 nginx 和 OpenSSL 这样的程序，但 VPS 的系统是 CentOS 6, 源用的是 EPEL，其中 OpenSSL 的版本是 1.0.1e-fips，而 OpenSSL 官方网站上带 -fips 的版本是 2.0了。故选择自己编译 OpenSSL，同时 nginx 也要一起重新编译。（源里面的 nginx 是 1.10.2, 是最新的稳定版，这比前一个 VPS 的源里的版本新的多了，不知道前面那个 VPS 自带的上游源没有同步，我当时也懒得换源）

2. 下载相应的源码，按依赖顺序编译

 这很简单，去官网找到链接，复制到 shell 里面用 wget 就好了。这里选择的是 nginx 1.10.2, OpenSSL 1.0.2j，PHP 7.1.0. 分别 `tar xf` 解开，按 OpenSSL -> nginx -> PHP 的顺序编译就好。`configure` 参数命令行如下所示：

```
# OpenSSL
./config --prefix=/root/root --prefix=/root/root/ssl

# nginx
./configure --prefix=/root/root --with-openssl=/root/src/openssl-1.0.2j

# PHP
./configure --with-openssl=/root/root/ssl --enable-mbstring --enable-fpm --prefix=/root/root --with-mysqli --with-pdo-mysql --with-gd
```

 每个 `configure` 结束以后用 `make install` 来安装到 /root/root 目录下，以避免污染系统自带的版本。库的版本不能随便乱动的，免得整个系统崩掉，更何况 OpenSSL 的 branch 都变了。要体验完全源码编译的乐趣的话，像 Gentoo 那样自动编译整个系统比较好……

 顺便吐个槽，PHP 的 `--with-openssl` 的帮助就只说了一个 Include OpenSSL support (requires OpenSSL >= 1.0.1)，谁知道它到底要的是 OpenSSL 的安装路径还是源码路径？还是 nginx 做得好，set path to OpenSSL library sources. 蛋疼的是，PHP 要求的路径是安装路径，与 nginx 相反……

 （为什么 Sublime Text 下载得这么慢啊）

3. 测试 nginx

 直接打 IP 进去，提示 403. 看起来是 nginx 默认自己降到 nobody 用户。为了方便管理，给它新增一个 nginx 专用账户。

```
groupadd nginx
useradd -g nginx nginx
chsh -s /sbin/nologin nginx
```

 最后一行是防止这个账户通过 shell 登录。（在 `useradd` 里面加两个参数就能用一条命令做完三条的事，不过我不记得了……）

 然后把 nginx 的 html 文件夹移动到 ~nginx/ 底下，`chown` 一下，403 就没有了。

4. 配置 & 测试 PHP

 这里有些麻烦。先是尝试了几次才找对全部需要的参数（如上命令行，`--enable-fpm` 和扩展相关的参数一开始就没加，得到的是裸的 PHP），然后复制 php-fpm.conf.default 把 .default 去掉，设置 socket 路径，改启动用户为 nginx，启动后手动 chown nginx:nginx 它的 socket 文件。nginx 有自带 PHP-FPM 的配置，一行 `include fastcgi.conf` 就好了。

 要特别注意的是 nginx 的 fastcgi_temp 目录需要指定到 nginx 能够写入的位置，我这里 nginx 的程序放在 ~root 下面，所以它默认的 temp 目录也在这下面。需要用 `fastcgi_temp_path` 指定到 ~nginx 底下去然后 chown，否则能返回 HTTP 200 但 php 网页显示不全。

 至于 PHP 的探针，`phpinfo();` 就足够了～

5. 微信推送测试

 由于公众号权限不足，故只能通过回复消息的形式给出链接，菜单功能只能稍后处理。回复消息包含的链接文本会自动转化成可点击的链接。


### 2016/12/28

今天主要的任务是写一个 Demo. 打算先实现简单的登录功能，然后显示点简单的数据，比如显示目前登录到的变电站名称，目前该站拥有的设备及其最近一次故障记录详情。自己的测试服务端里要写死以上数据。

任务书还没下来，不知道要做微信登录还是用户名密码登录。微信登录个人感觉不是很必要，因为我们这里并不强调人与人的关系。所以，这个 Demo 还是按用户名和密码做。

此外有一点要注意的地方，就是这里面涉及的数据都比较私密，所以前端发送密码时应该加盐 hash 一下，并且做到每次发送的字符串都不一样。但是查了一下关于这个问题的讨论 [1]，发现还是有不少人认为只要有 HTTP 连接就没有加密的意义了，即便是一次一密，也能通过前端页面公开的算法甚至 XSS 做中间人攻击。但是我觉得如果服务器真的是在登录的时候发来 nonce，那么即便是监听到了也很难通过 hash 反推密码啊。一个 nonce 怎么也要十几个字符了，加用户名加密码得有二三十个字节长，用 bcrypt 之类的怎么反推得了……（当然 HTTPS 是最好了，但目前要上 HTTPS 的话要通过 CloudFlare，速度会很慢）

[1] https://www.zhihu.com/question/25539382

做的时候发现 Javascript 里似乎没有现成的散列函数，选择用 SHA256 [2] 好了。

[2] http://www.bichlmeier.info/sha256.js

PHP 是被访问时执行一次然后退出，以前以为是必须要把当前会话的状态存入数据库的，现在查到 PHP 自身支持 session 功能，不需要数据库也可以在不同的访问中保留状态。

（发现 PHP 已经忘得一干二净了）

（为什么 PHP 在使用全局变量的时候还要先用 `global` 啊）

HTML 从头开始写的话有点懵，最后从 MDN 找了 AJAX 请求代码修改了下来验证 SHA256 函数以及登录过程实现正确。最后是验证成功了，今天就到这里了，代码已经 push 到 GitHub 了，有心情再回去看吧。

（说起来，总觉得 PHP 里面的那些类实现得很奇怪，突然不知道类里应该要写哪些方法了……）

### 2016/12/29

今天要汇报了，想着给那个测试登录用的网页加点样式吧。不过这次时间有点赶，看书可能来不及，先口胡两句 CSS 上去……

CSS 水平居中好说，垂直居中的话如果用父元素 table、子元素 table-cell 的方法，要记得在 html {} 里就把 `height: 100%` 写好，否则后面 inherit 来的高度最大也就只有那么大了，不能覆盖全屏。

剩下的都慢慢调……不知道为什么 `position: absolute; left: 0; right: 0;` 的 copyright notice居然没有跟登录框水平对齐，明明登录框已经是 `text-align: center` 了。好像是登录框歪了，不知道为什么……暂时把 left 改成 2%，看起来舒服一点。
