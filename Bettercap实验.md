# Bettercap

BetterCap是一个功能强大、模块化、轻便的MiTM框架，可以用来对网络开展各种类型的中间人攻击（Man-In-The-Middle）。它也可以帮你实时地操作HTTP和HTTPS流量，以及其他更多功能。它也被称为EtterCap的继任者

## 特点

- 全双工和半双工的ARP欺骗

- 首次实现了真正的ICMP双向欺骗

- 可配置的DNS欺骗

- 实时和完全自动化地主机发现

- 实时获取通信协议中的安全凭证，包括HTTP(s)中的Post数据，Basic和Digest认证，FTP、IRC、POP、IMAP、SMTP、NTLM(HTTP、SMB、LDAP，etc)以及更多

- 完全可定制的网络嗅探器

- 模块化的HTTP和HTTPS透明化代理，支持将用户以及内置的插件注入到目标的HTML代码、JS或CSS文件，以及URLs中

- 使用HSTS bypass技术拆封SSL

- 内置HTTP服务器

## Bettercap V.S. Ettercap

EtterCap有一些附加功能，比如你可以看到连接状态，原始的pcap包。但是这些并不是对所有人都有用。Ettercap有点陈旧了，BetterCap没有以下弊端：

- Ettercap是一个伟大的工具，但是有点老了

- Ettercap的过滤功能大多数情况下无效

- Ettercap的过滤功能过时了

- Ettercap过滤器很难实现，这与实现它的语言有关

- Ettercap在大型网络中不稳定（在比/24大的网络中尝试主机发现时）

- Ettercap很难扩展或者添加模块，除非你是一个C/C++开发人员

- Ettercap和MITMF的ICMP欺骗功能在2016年完全没用

- Ettercap不支持内置或者模块化的HTTP(s)透明代理

- Ettercap不提供只能的可定制的安全凭证嗅探器

## 实验

### 实验环境

- 攻击者 drone

    - eth0 NAT网络 10.0.2.7

-  靶机 ns

    - eth0 NAT网络 10.0.2.5

- 网关 10.0.2.1

- Bettercap 

    - v2.1.1

    - v1.6.2

![](img/Bettercap/networks_topolpgy.png)

### v2.1.1版本安装

v2.1.1采用的是嵌入式的终端页面形式，v2.0以上版本改用Go语言编写，直接使用`apt-get`安装即可

    apt-get update
    apt-get install bettercap
    bettercap

查看当前有哪些服务正在运行

    help
    # events.stream 将事件打印为连续流 
    # net.recon 周期性查看 arp 缓存，用于管理网络中的新主机 

![](img/Bettercap/help.png)

### 尝试ARP欺骗

arp欺骗前的靶机arp缓存表

![](img/Bettercap/beforearp.png)

在bettercap v2 中，如果不进行设置的话，arp欺骗是默认整个子网的，所以最好进行设置一下

    set arp.spoof.internal true
    set arp.spoof.targets 10.0.2.5
    set arp.spoof.verbose false

    net.sniff on
    arp.spoof on

![](img/Bettercap/afterarp.png)

登陆某些没有使用https协议并且明文传输密码的网站，可以获得用户名和密码等相关敏感信息

![](img/Bettercap/getpw.png)

### 开启http和https代理，尝试进行https劫持

输入如下指令

    net.probe on
    http.proxy on
    https.proxy on
    arp.spoof on
    # 发现整个子网只有我的实验机，偷懒就进行了整个子网的arp欺骗

http网站进行访问，并截获交换信息。但如果访问`github.com`或`baidu.com`等经过https加密过的网站，浏览器会阻止访问，提示连接不安全

![](img/Bettercap/https.png)

### 尝试进行ssl剥离

在代理开启时，sslstrip是默认关闭的，我们进行手动开启

    set http.proxy.sslstrip true
    set https.proxy.sllstrip true
    set arp.spoof.internal true
    set arp.spoof.targets 10.0.2.5
    set net.sniff.verbose false

    net.sniff on
    http.proxy on
    https.proxy on
    arp.spoof on

但不知道为什么访问https的网站仍然无法建立连接，查阅了相关资料貌似是说bettercap v2功能不稳定的原因，决定换为v1版本继续进行实验

### 卸载v2版本，下载v1版本

v1版本的仓库已经被弃用，只能从其他用户的仓库中下载

    # 下载源码
    git clone https://github/sqqihao/bettercap.git
    # 安装依赖
    sudo apt-get install build-essential ruby-dev libpcap-dev net-tools
    # 删除用apt-get下载的v2版本
    apt-get remove bettercap
    # 编译
    cd bettercap
    gem build bettercap.gemspec
    sudo gem install bettercap*.gem

### 再次尝试ssl劫持

    bettercap --proxy-https -T 10.0.2.5

ssl劫持和之前v2版实验一样，无法建立连接，会进行证书校验

![](img/Bettercap/SSL0.png)

### 再次尝试SSL剥离并进行HTTPS数据重定向

    bettercap -T 10.0.2.5 --proxy 

访问百度，但没有成功剥离，显示的仍是带有https的页面，多次尝试仍是如此

访问12306成功进行了SSL剥离，HTTPS网址被替换`wwww.12306.cn`

![](img/Bettercap/12306.png)

尝试对alibaba进行ssl剥离，在剥离前进行访问返回的是这样的页面

![](img/Bettercap/alibababefore.png)

进行中间人攻击后返回的网站，网址由`https://www.alibaba.com`变成了`www.alibaba.com`，且页面显示一样，不仔细看难以发现差别

![](img/Bettercap/alibaba.png)

想到不可能12306和alibaba会遇到ssl剥离，而百度不会，于是清除了浏览器的所有缓存和历史记录，再去访问百度，此时成功ssl剥离，网址变为`wwww.baidu.com`

![](img/Bettercap/baidu.png)

对163邮箱进行测试，获得Cookie，存在被重放攻击的可能性

![](img/Bettercap/163-1.png)

对qq邮箱测试，会不断进行重定位，无法显示页面

![](img/Bettercap/qq.png)

想起来之前通过百度搜索163，百度会直接在搜索页面显示快捷登陆框以方便用户直接进行登录，包括360搜索也提供类似的服务

![](img/Bettercap/163-2.png)

当ssl被剥离时，使用该方式进行163邮箱登陆，查看截获的数据，发现了用户名的用户名、密码和Cookie

![](img/Bettercap/163-3.png)

## 参考

- [BetterCap：EttherCap的继任者](https://github.com/CUCCS/ns/blob/2016-2/2016-2/zzj_cay/TASK.Exploration_and_experiments_on_SSL_MITM_attacks/%E6%8B%93%E5%B1%95%E5%AE%9E%E9%AA%8C2.md)

- [bettercap](https://www.bettercap.org/)

- [中间人攻击利用框架bettercap测试](http://www.ijiandao.com/safe/cto/15365.html)

- [Bettercap v2 Kullanımı](https://www.youtube.com/watch?v=XHpApXKJMqs)

- [瑞士军刀 Bettercap2.4使用教程](https://blog.csdn.net/u012570105/article/details/80561778)

- [BetterCap：一个模块化的轻便的MiTM框架](https://www.freebuf.com/sectool/99607.html)

- [新版本的bettercap不好用， 如何安装和编译旧版本的bettercap](https://www.cnblogs.com/diligenceday/p/9912542.html)