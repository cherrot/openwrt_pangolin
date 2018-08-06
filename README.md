**openwrt_pangolin**是一个快速部署路由器透明代理（路由器翻墙）的配置集，适用已刷入[OpenWRT/LEDE](https://openwrt.org/)固件的便携和家用路由器。

1. [功能](#features)
2. [原理](#explanation)
    1. [DNS污染](#dns-pollution)
    2. [IP屏蔽/TCP阻断](#ip-blocking)
3. [安装](#installation)
4. [配置说明](#config)
    1. [工作模式相关](#work-mode)
    2. [DNS相关](#dns-related)
    3. [端口转发相关](#port-forwarding)
5. [已测试设备](#tested-devices)
6. [参考](#refs)

<a name="features"></a>
功能
=====

本项目提供的所有功能均为可选配置，根据个人需要拷贝相关配置即可。具体的配置文件说明见“配置说明”一节。

* 透明代理：终端无需任何设置即可访问国际互联网。代理工具支持[shadowsocks-libev][ss-libev]和[v2ray][v2ray]。
* 智能分流：国外HTTP/HTTPS流量走代理，国内流量、BT流量直连。
* 抵御DNS污染。
* 可以设置三种工作模式，轻触`reset`切换模式：
    + AP模式：用网线接到上级路由出口，比较适合在酒店使用。
    + client模式：使用无线连接到上级路由，适合没有网线不适宜使用AP模式的场合。网速会比AP模式稍慢。
    + router模式：适合需要拨号上网的情况，比如家庭路由器。
* 子网段为`10.10.10.0/24`，不太容易跟上级路由冲突。

<a name="explanation"></a>
原理
=====

<a name="dns-pollution"></a>
DNS污染
-------

目前GFW的DNS污染只作用于UDP协议，理想中只要使用一个未被污染且支持TCP查询的DNS即可。
然而大多数比较知名的公共DNS的IP都已被GFW阻断，无法连接，因此我们采取两个应对方案：

1. 使用`DNScrypt`等安全DNS方案获取可靠的查询结果，因为较为小众所以尚未被GFW阻断；
2. 使用代理转发DNS查询获取未被污染的结果。

我们使用`unbound`作为DNS服务器，因为其配置简单，策略灵活。
默认设置国内常用的音视频网站域名通过DNSPod公共DNS查询，因为内容网站大多会根据查询来源返回就近的服务器，以提高访问速度。
其他域名应用上述策略获得未污染结果。由于`unbound`有DNS缓存，并通过开启`prefetch`有效防止缓存过期，因此DNS查询速度对网络体验的影响可以忽略。

<a name="ip-blocking"></a>
IP屏蔽/TCP阻断
--------------

对抗手段就是VPN或者代理，把流量通过尚未被屏蔽的IP转发出去。本项目默认支持两种代理工具：

1. [shadowsocks-libev][ss-libev]：配置简单，体积小巧，功能强大。使用`shadowsocks`协议实现安全数据传输。
2. [v2ray][v2ray]：配置非常灵活，支持多种协议，并可以灵活搭配，实现协议伪装等需求。
    但其配置文件略长，有一定学习门槛，并且软件体积较大（6~10MB），内存占用较多（>50MB）。

除此之外，我们使用`iptables`设置智能分流，只有到国外IP的HTTP/HTTPS流量经过代理。
IP规则用了[精简过的国内IP段][china-list-gist]。如果需要自己生成，可以执行`gen_china_list.sh`生成`chiana.list`，自行精简后拷贝到`/root/config/`。

<a name="installation"></a>
安装
=====

1. 安装基础依赖，通过LuCI页面安装或者`ssh root@192.168.1.1`登录路由器执行命令（假定路由器IP`192.168.1.1`）：

    ``` shell
    opkg install unbound ipset dnscrypt-proxy ca-certificates
    ```

    **注意：** 如果不打算使用`dnscrypt`，可以只安装`unbound`和`ipset`；如果使用`v2ray`并且启用了TLS，那么仍需要安装`ca-certificates`。

2. 下载安装代理工具：
    - [shadowsocks-libev][openwrt-ss]：根据路由器CPU架构下载对应的包，在路由器上执行`opkg install ./shadowsocks-libev_VERSION_ARCH.ipk`即可安装。
    - [v2ray][v2ray-release]：根据CPU架构下载对应的包到本地，解压后上传到`/root/v2ray`。如需精简体积，可尝试[自行编译][build-v2ray]。

    **注意：** `v2ray`根据配置不同，内存占用在20M~100M不等，内存少于128MB的路由器不建议使用。

3. 配置代理工具：编辑`etc/shadowsocks.json`或`root/v2ray.json`。不建议修改本地端口（默认 **1080**），否则`firewall.user`中也要做出相应修改。

4. 编辑`etc/firewall.user`， **将PROXY_SERVER_IP替换为代理服务器IP**，如有多个使用逗号分隔。

5. （可选）编辑`etc/config/network.router`， **填入你从ISP获得的拨号账号与密码**

6. （可选）编辑`etc/config/wireless.{ap,router,client}`，更改WIFI SSID和密码，加密模式可以参考[OpenWRT Wiki][wpaencryption]。
    默认为 **穿山甲@[模式]**，密码均为 **88888888**。可以把不同模式下的SSID都设成一样，但是这样会不太方便区分路由器运行在什么模式下。

7. （建议）备份路由器配置。ssh登录路由器，备份`/etc`目录：

    ``` shell
    tar -czvf etc.tar.gz /etc
    ```

8. 拷贝需要的配置文件到OpenWRT路由器，注意要用`-p`参数保留文件权限。以下示例拷贝了全部配置：

    ``` shell
    scp -rp etc root root@192.168.1.1:/
    ```

9. ssh登录路由器，确保`firewall`, `unbound`, `shadowsocks` / `v2ray`在启动时自动运行：

    ``` shell
    /etc/init.d/firewall enable
    /etc/init.d/unbound enable
    # shadowsocks和v2ray只能启用一个
    /etc/init.d/shadowsocks enable
    /etc/init.d/v2ray enable
    ```

10. 重启动路由器，可在LuCI页面控制或者登录路由器执行`reboot`。

<a name="config"></a>
配置说明
========

<a name="work-mode"></a>
工作模式相关
-----------

1. `etc/dnsmasq.conf`：设置了DHCP的IP分配范围，从`10.10.10.100`到`10.10.10.150`。

2. `etc/config/network*`和`etc/config/wireless*`：分别是不同工作模式下的有线和无线配置。轻触`reset`时，会切换`/etc/config/network`和`/etc/config/wireless`的软连接指向并重启网络接口，从而实现工作模式的切换。
如果对里面的具体配置有疑惑，可以先从路由器LuCI页面上配置好网络，保存后与本项目的网络配置文件作比较。

3. `etc/rc.button/reset`：设置`reset`按钮，通过调用`/root/bin/toggle_ap.sh`切换工作模式。

4. `root/bin/`：`toggle_ap.sh`负责切换工作模式；`auto_toggle.sh`可以设置为开启启动脚本，如果当前工作模式为client但扫描不到名为“XXXclient”的无线网时，自动切换工作模式，这样可以确保路由器能够启用无线网络。

<a name="dns-related"></a>
DNS相关
-------

1. `etc/config/dhcp`: 设置`dnsmasq`将DNS查询转发到`unbound`监听的1053端口。主要改动如下：
    ``` conf
    option noresolv '1'
    list server '127.0.0.1#1053'
    list server '::1#1053'
    ```

2. unbound相关配置，主要改动如下：
    - `etc/config/unbound`: unbound UCI config, 从LEDE 17.01和OpenWRT 18.06开始提供。
        ``` conf
        option listen_port '1053'   # 设置监听端口1053
        option manual_conf '1'      # 要求加载 /etc/unbound/unbound.conf
        ```

    - `etc/unbound/unbound.conf`: unbound标准配置，为了向后兼容重复了部分设置。
        ``` conf
        server:
            num-threads: 2      # unbound线程数，建议与路由器CPU核数一致
            so-reuseport: yes   # 线程数>1时开启
            port: 1053
            tcp-upstream: yes   # 使用TCP请求
            # 向所有DNS服务器发送edns-client-subnet，优化解析结果
            send-client-subnet: 0.0.0.0/0  # 需要unbound >= 1.6.7

        # 读取china-dns配置，对国内音视频网站使用DNSPod解析
        include: "/etc/unbound/china-dns.conf"

        # 其他域名使用DNScrypt/代理转发
        forward-zone:
              name: "."
              # DNScrypt upstream
              forward-addr: 127.0.0.1@5353
              # Public DNS servers
              forward-addr: 8.8.8.8
        ```

    - `etc/unbound/china-dns.conf`: 可以自行添加国内域名解析规则。
        ``` conf
        forward-zone:
            name: "qq.com."
            forward-addr: 119.29.29.29
        ```
3. `etc/config/dnscrypt-proxy`: DNScrypt配置，默认监听5353和5454两个端口。如果不使用需要更新`unbound.conf`的转发地址。

<a name="port-forwarding"></a>
端口转发相关
------------

1.  `etc/firewall.user`和`root/config/china.list`：设定`iptables`转发规则，做了一点微小的工作：
    1. 创建一个名为"BREAKWALL"的NAT规则；
    2. 忽略目的地址为代理服务器的IP数据包；
    3. 创建`ipset`表，设置忽略内网地址和来自`china.list`的国内IP段；
    4. 其他路由到此规则的TCP流量统统转发到 **1080**端口
    5. 将该规则应用到所有目标端口为80和443的TCP流量（HTTP和HTTPS流量）和从路由器发出的TCP DNS查询流量。

    **注意：** 如果你的转发规则没有生效，检查`/etc/config/firewall`，确保包含如下配置：
    ``` conf
    config include
        option path '/etc/firewall.user' 
    ```
2.  `etc/init.d/`包含了代理服务自启动配置；代理工具配置文件在`root/config/`下。

**关于UDP转发：** `iptables`的`REDIRECT`是不能成功转发UDP流量的，原因貌似是无法正确得到UDP的原始目的地址。因此对于DNS流量的转发，网络上流行的方案是使用`TPROXY`，实现UDP流量的透明代理。正如`firewall.user`文件最后被注释掉的部分那样。而我最终选择用`unbound`派发TCP DNS请求，并由`iptables`转发至代理的方案，理由如下：

- OpenWRT默认裁剪了`TPROXY`模块，需要额外安装用户空间模块`iptables-mod-tproxy`才能支持。转发UDP流量时会带来内核态到用户态的切换，有一定的性能开销；而且`iptables`规则也会更加复杂。
- 与路由器到代理服务器的耗时相比，代理服务器到DNS服务器的TCP连接开销完全可以忽略不计；

**Tips:** 如果你需要添加其他DNS，可以在墙外节点用下面的命令测试是否支持TCP查询：
``` shell
dig @8.8.8.8 +tcp twitter.com
```

<a name="tested-devices"></a>
已测试设备
==========

欢迎提交issue补充更多设备支持信息。

便携路由：

[Nexx WT3020](https://wiki.openwrt.org/toh/nexx/wt3020), [TP-Link TL-WR703N](https://wiki.openwrt.org/zh-cn/toh/tp-link/tl-wr703n), [Kingston MLWG2](https://wiki.openwrt.org/toh/kingston/mlwg2)

家用路由：

[Linksys WRT1900ACS](https://openwrt.org/toh/linksys/wrt_ac_series#wrt1900acs)

<a name="refs"></a>
参考
====

- [内核透明代理模块TPROXY](https://www.kernel.org/doc/Documentation/networking/tproxy.txt)
- [shadowsocks-libev透明代理设置][ss-tproxy]
- [unbound中不同forward-addr的选择策略][unbound-forward]


[ss-libev]: https://github.com/shadowsocks/shadowsocks-libev
[v2ray]: https://www.v2ray.com/
[v2ray-release]: https://github.com/v2ray/v2ray-core/releases
[build-v2ray]: https://steemit.com/cn/@v2ray/meemg-v2ray
[china-list-gist]: https://gist.github.com/zts1993/dca7c062a520396d3091
[openwrt-ss]: https://github.com/shadowsocks/openwrt-shadowsocks/releases
[ss-tproxy]: https://github.com/shadowsocks/shadowsocks-libev#transparent-proxy
[wpaencryption]: https://openwrt.org/docs/guide-user/network/wifi/basic#wpa_modes<Paste>
[openwrt-netfilter]: https://openwrt.org/docs/guide-user/firewall/netfilter-iptables/netfilter
[unbound-forward]: https://nlnetlabs.nl/pipermail/unbound-users/2018-January/005054.html
