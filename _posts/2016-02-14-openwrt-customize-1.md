---
layout: post
title: 定制 OpenWrt 固件 (1) 透明代理
categories:
  - Small Things
tags:
  - openwrt
  - lede
  - router
---

一年一度虐狗节 :) 今年谁这么无聊，你给我出来，我保证不打死你 :)

折腾 OpenWrt 也有些日子了，不过还从来没尝试过自己定制 Rom。今天心血来潮，想把手里目前闲置的一个路由拿去做个中继 + 透明代理。

## Repeater

现在市面上能买到的很多路由器都支持中继模式，我手里这台 HG556a ver.C 在更新到 OpenWrt Chaos Calmer 正式版之后居然也支持了。实测中继模式运行完美，美滴很。

本来我还不得不插上一个 Atheros 的无线网卡进行中继呢，现在倒是省事了。

## Transparent Proxy

所谓透明代理，就是说，用户完全感受不到代理的存在，但是又能享受代理提供的网络服务。

实际使用时，用户不需要设置代理或者 VPN，就能够达到翻墙等目的。

<!-- more -->

## OpenWrt ImageBuilder

这次我们的定制选择使用 OpenWrt ImageBuilder 进行。因为 Build 的是 Chaos Calmer 正式版映像，没有必要从源代码开始编译。如果是 Trunk 的话，由于内核版本迭代非常快，建议从源代码开始编译。

首先下载 Chaos Calmer 的 ImageBuilder。官方源很不稳定而且速度慢，推荐中科大的镜像源：http://mirrors.ustc.edu.cn/openwrt/

下载时只需确定自己的设备类型就可以，比如我们这次使用的 HG556a ver.C 属于 brcm63xx。

还有一点，brcm63xx 以及很多其他的 BSP 都有 generic 和 smp 两种固件。这是因为 BroadCom 有几个 CPU 为双核心，但是其架构比较特别，比如我们使用的 HG556a 的 BCM6358，两个核心共享同一个 TLB，因此 OpenWrt 官方并不支持在 BCM6358 SoC 上使用双核心。而 BCM6362 和 BCM6368 每个核心拥有独立的 TLB，在这些 SoC 上使用 smp 固件就可以完全释放双核心的潜能。

这里还有一点要说明，BCM6358 的两个核心并不完全相同，Core 0 的 Instruction Cache 比 Core 1 的大一倍，然而 OpenWrt 默认只使用 Core 1。我们可以通过修改 CFE 中部分代码，使系统使用 Core 0。据说可以获得 15-20% 的性能提升。

我们在这里选 generic 和 smp 没有太大区别，干脆也选 smp 吧。那么下载得到的文件就是 OpenWrt-ImageBuilder-15.05-brcm63xx-smp.Linux-x86_64.tar。

解压之后我们可以看到下面这些东西：

{% highlight bash %}
$ ll
总用量 1068
drwxr-xr-x 11 yichya yichya   4096  2月 14 01:38 ./
drwxrwxr-x  4 yichya yichya   4096  2月 13 16:14 ../
drwxrwxr-x  3 yichya yichya   4096  2月 13 15:17 bin/
drwxr-xr-x  3 yichya yichya   4096  9月  4 21:37 build_dir/
-rw-r--r--  1 yichya yichya  98530  9月  4 21:37 .config
drwxrwxr-x  2 yichya yichya  12288  2月 13 23:04 dl/
drwxr-xr-x  3 yichya yichya   4096  9月  4 21:38 include/
-rw-r--r--  1 yichya yichya   5239  7月 24  2015 Makefile
-rw-r--r--  1 yichya yichya 686463  9月  4 21:13 .packageinfo
drwxr-xr-x  3 yichya yichya   4096  2月 13 15:17 packages/
-rw-r--r--  1 yichya yichya   1211  2月 13 16:57 repositories.conf
-rw-r--r--  1 yichya yichya  10680  7月 24  2015 rules.mk
drwxr-xr-x  4 yichya yichya   4096  7月 24  2015 scripts/
drwxr-xr-x  3 yichya yichya   4096  9月  4 21:37 staging_dir/
drwxr-xr-x  3 yichya yichya   4096  9月  4 21:37 target/
-rw-r--r--  1 yichya yichya 223540  9月  4 21:13 .targetinfo
drwxrwxr-x  2 yichya yichya   4096  2月 13 15:15 tmp/
{% endhighlight %}

首先我们需要做的事情是，配置 repositories.conf，将 OpenWrt.org 官方源换为 ustc.edu.cn 提供的镜像源。

{% highlight bash %}
## Place your custom repositories here, they must match the architecture and version.
# src/gz chaos_calmer http://downloads.openwrt.org/chaos_calmer/15.05/brcm63xx/smp/packages
# src custom file:///usr/src/openwrt/bin/brcm63xx/packages

## Remote package repositories
src/gz chaos_calmer_base http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/base
src/gz chaos_calmer_luci http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/luci
src/gz chaos_calmer_packages http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/packages
src/gz chaos_calmer_routing http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/routing
src/gz chaos_calmer_telephony http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/telephony
src/gz chaos_calmer_management http://mirrors.ustc.edu.cn/openwrt/chaos_calmer/15.05/brcm63xx/smp/packages/management

## This is the local package repository, do not remove!
src imagebuilder file:packages
{% endhighlight %}

这样，后面生成映像的速度会有很大提升。

输入 make 命令查看帮助：

{% highlight bash %}
$ make
Available Commands:
	help:	This help text
	info:	Show a list of available target profiles
	clean:	Remove images and temporary build files
	image:	Build an image (see below for more information).

Building images:
	By default 'make image' will create an image with the default
	target profile and package set. You can use the following parameters
	to change that:

	make image PROFILE="<profilename>" # override the default target profile
	make image PACKAGES="<pkg1> [<pkg2> [<pkg3> ...]]" # include extra packages
	make image FILES="<path>" # include extra files from <path>
	make image BIN_DIR="<path>" # alternative output directory for the images
    
{% endhighlight %}

那么我们先看一下这个包中包含的所有 target profile 吧。

{% highlight bash %}
$ make info
Current Target: "brcm63xx (smp)"
Default Packages: base-files libc libgcc busybox dropbear mtd uci opkg netifd fstools swconfig kmod-gpio-button-hotplug dnsmasq iptables ip6tables ppp ppp-mod-pppoe kmod-nf-nathelper firewall odhcpd odhcp6c
Available Profiles:

Default:
	Default Profile
	Packages: kmod-b43 wpad-mini
963281TAN:
	Generic 963281TAN
	Packages: 
96328avng:
	Generic 96328avng
	Packages: 

# ...

HG556a_AB:
	Huawei EchoLife HG556a (version A/B - Atheros)
	Packages: kmod-ath9k wpad-mini kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev
HG556a_C:
	Huawei EchoLife HG556a (version C - Ralink)
	Packages: kmod-rt2800-pci wpad-mini kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev 
HG655b:
	Huawei HG655b
	Packages: kmod-rt2800-pci wpad-mini kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev

# ...
{% endhighlight %}

我们看到 HG556a ver.C 对应的 profile 是 HG556a_C。那么我们直接试着 make image 一下吧。

{% highlight bash %}
$ make image PROFILE="HG556a_C"
{% endhighlight %}

没费多少功夫就结束了，然后我们就可以在 bin 文件夹下面找到这些东西：

{% highlight bash %}
$ ll bin/brcm63xx/
总用量 8504
drwxrwxr-x 2 yichya yichya    4096  2月 14 19:25 ./
drwxrwxr-x 3 yichya yichya    4096  2月 13 15:17 ../
-rw-rw-r-- 1 yichya yichya     292  2月 14 16:17 md5sums
-rw-rw-r-- 1 yichya yichya 4980740  2月 13 23:04 openwrt-15.05-brcm63xx-smp-HG556a_C-squashfs-cfe.bin
-rw-rw-r-- 1 yichya yichya 3801088  2月 14 16:17 openwrt-15.05-brcm63xx-smp-root.squashfs
-rw-rw-r-- 1 yichya yichya     452  2月 14 16:17 sha256sums
{% endhighlight %}

其中 squashfs 文件是根文件系统的映像，而 bin 文件就是可以直接刷到路由器里面的固件了。

当然，这个固件的功能非常简单，简单到连 Web 界面都没有。想要设置只能通过 ssh 连接到路由，通过手动修改 /etc/config 下的配置文件完成对路由器的设置。

## Get Started

上一步中我们定制的固件功能缺乏，以至于很难满足普通用户的正常使用需求。因此，我们需要增加固件中包含的软件包数量。

我们在 make info 的时候可以看到最上面的 Default Packages 和下面每一个 target 对应的 profile 中写明的 Packages。从配置文件中看，HG556a_C 中只包含了最基础的包、Ralink 无线网卡驱动、USB 支持和一个 LED 驱动，并不包括 Web 界面。

在我们这次的定制中，我们希望加上图形界面，然后再添加一些例如 U 盘支持、SFTP、动态 DNS 等的功能。于是我们选定所有需要的包：

* wget 
* luci 
* block-mount 
* openssh-sftp-server
* kmod-fs-vfat
* kmod-fs-ext4 
* luci-app-ddns

然后，修改配置文件。配置文件的位置在 /target/linux/brcm63xx/profiles 目录下。在里面我们可以找到一个 huawei.mk 文件，跟我路由器相关的配置应该就在里面了。

打开 huawei.mk，果然找到了 HG556a_C 的配置：

{% highlight makefile %}
define Profile/HG556a_C
  NAME:=Huawei EchoLife HG556a (version C - Ralink)
  PACKAGES:=kmod-rt2800-pci wpad-mini \
	kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev
endef
define Profile/HG556a_C/Description
  Package set optimized for Huawei HG556a version C (Ralink).
endef
$(eval $(call Profile,HG556a_C))
{% endhighlight %}

只要在中间的 PACKAGES 节中增加我们需要的包就可以了。在这里我们无需人工处理依赖，Imagebuilder 会自动帮我们完成依赖相关的工作。

修改成下面这样：

{% highlight makefile %}
define Profile/HG556a_C
  NAME:=Huawei EchoLife HG556a (version C - Ralink)
#  PACKAGES:=kmod-rt2800-pci wpad-mini \
#	kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev
  PACKAGES:= kmod-rt2800-pci wpad-mini kmod-usb2 kmod-usb-ohci kmod-ledtrig-usbdev wget luci block-mount openssh-sftp-server kmod-fs-vfat kmod-fs-ext4 luci-app-ddns
endef
define Profile/HG556a_C/Description
  Package set optimized for Huawei HG556a version C (Ralink).
endef
$(eval $(call Profile,HG556a_C))
{% endhighlight %}

保存，然后回到主目录，生成映像：

{% highlight bash %}
$ make image PROFILE="HG556a_C"
{% endhighlight %}

很快就得到了一个我们定制好的映像。

## Add repositories from blocked sources

准备透明代理则稍微复杂了那么一点儿。参考来自 <https://cokebar.info/archives/664> 的教程，我们需要再安装以下几个包：

* ip 
* ipset
* libopenssl
* iptables-mod-tproxy

以及来自第三方的

* shadowsocks-libev-spec
* ChinaDNS
* luci-app-shadowsocks
* luci-app-chinadns

将上面几个软件包添加到 target profile 中就可以实现自动安装了。

不过，麻烦的事情在于，第三方的源是直接被 GFW 拉黑的 sourceforge。

在 repositories.conf 中添加以下两个源：

{% highlight bash %}
# ShadowSocks
src/gz openwrt_dist http://openwrt-dist.sourceforge.net/releases/brcm63xx/packages
src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/releases/luci/packages
{% endhighlight %}

直接 make image 的话，在从上面两个源更新列表时会卡住很久，不得不按 Ctrl + C 终止。

{% highlight bash %}
Downloading http://openwrt-dist.sourceforge.net/releases/brcm63xx/packages/Packages.gz.
^CMakefile:100: recipe for target '_call_image' failed
make[1]: *** [_call_image] Interrupt
Makefile:178: recipe for target 'image' failed
make: *** [image] 中断
{% endhighlight %}

在这里为了解决在终端下使用代理的问题，我们需要使用一个叫做 proxychains 的工具，它可以支持 SOCKS5 全局代理。

安装 proxychains：

{% highlight bash %}
sudo apt-get install proxychains
{% endhighlight %}

修改 proxychains 的配置文件 /etc/proxychains.conf

{% highlight bash %}
strict_chain
proxy_dns 
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
quiet_mode

[ProxyList]
socks5  127.0.0.1 1080
{% endhighlight %}

然后在 make image 时使用这样的命令：

{% highlight bash %}
$ proxychains make image PROFILE="HG556a_C"
{% endhighlight %}

## Configure Transparent Proxy

制作好了能够完成透明代理的固件，我们还需要进行一些配置，才能顺利的实现透明代理。

### Enable ShadowSocks

在服务页面中选择 ShadowSocks，填写好基本配置。

![shadowsocks-1](../assets/images/openwrt-customize-1/ss-1.png)

### Enable ChinaDNS

ChinaDNS 的配置有以下几种可选方案：

* 直接在墙外的远程主机上搭建 DNS 缓存服务器，使用其他的端口。
* 利用 ShadowSocks 转发 UDP 53 端口。
* 直接连接到外部 DNS 服务器，在 DNS 查询时不利用 ShadowSocks。
* 利用 TCP 方式完成 DNS 查询。

对于各种方式的优缺点，引用原文：

> 前两种方案可以有效避免 GFW 造成的 DNS 污染，此时 ChinaDNS 的作用完全就是为国外站点做解析优化，因为 DNS 查询由代理服务器转发，DNS 查询的发起者相当于代理服务器，获得的解析结果都是就近于代理服务器的；
>
> 而方案三则会直面 DNS 污染，此时 ChinaDNS 的防污染功能发挥作用，同时方案三无法做到国外网站解析结果优化，因为获得的 IP 是就近于你的位置的。
>
> 方案四则可以有效的避免国外 DNS 的污染，不过同样也无法做到国外网站解析结果优化。

而且，方案四还有一个严重的问题：慢。

我的 ShadowSocks 服务器并没有搭建 DNS 缓存（我也搭不了，不是我的），但是支持 UDP 端口转发，因此，我们选择第二种方案。

在服务页面中选择 ShadowSocks，填写好 UDP 端口转发配置，将 UDP 5300 转发到 Google 公共 DNS 即可。

![shadowsocks-2](../assets/images/openwrt-customize-1/ss-2.png)

然后配置 ChinaDNS，设置上游 DNS 为一个国内 DNS 和我们转发的 DNS（即 127.0.0.1:5300）。Local Port 就是 ChinaDNS 提供服务的端口，这里默认 5353 即可。Bidirectional Filter 建议打开，可以减少受到 DNS 污染的几率，但是可能会减慢部分网站访问速度。

![chinadns](../assets/images/openwrt-customize-1/chinadns.png)

### Configure DnsMasq

最后配置 dnsmasq 将所有的 DNS 请求转发给 ChinaDNS 的 5353 端口。

![dns](../assets/images/openwrt-customize-1/dns.png)

确定 ChinaDNS 和 ShadowSocks 都已经启动并正常工作，即可享受透明代理带来的愉悦啦。

## Speed Test

我们利用国内外两个网速测试工具进行速度测试。

国内的网站可以自动识别到国内，不经过代理，不影响速度。

![speedtest-inside](../assets/images/openwrt-customize-1/speedtest_inside.png)

国外的网站自动通过代理访问，实现自动翻墙。

![speedtest-outside](../assets/images/openwrt-customize-1/speedtest_outside.png)

同时可以看到，整个过程中我并没有打开右上角的 SwitchyOmega 代理切换。

在使用中还会发现一点小问题：比如说访问京东的时候，网站就会认为我的请求来自国外，然后在页面顶端显示一个导航栏建议我到京东的国外站点购物，真是够了 233333

顺便预告一下：下一次介绍集成 ADBYBY 和 KMS 服务器，当然下一次我会换新买的 7620n SoC 的路由器来演示；之后我会以集成石像鬼 QoS 为例介绍一下 OpenWrt 的 BuildRoot。