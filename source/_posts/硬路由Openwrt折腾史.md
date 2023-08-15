---
title: 硬路由 OpenWrt 折腾史
---

# 硬路由 OpenWrt 折腾史

## 前言

1. 什么是 OpenWrt?

借用官网的一句话，OpenWrt 项目是一个针对嵌入式设备的 Linux 操作系统。在本次的介绍中，我们将 OpenWrt 安装到路由器上，让路由器发挥更强大的功能

2. 为什么用折腾？

一般家里面的路由器基本上就一个功能：上网。但是路由器作为家里所有联网设备与外部的 Internet 交互的网关，应该可以发挥更强大的功能。OpenWrt 是一个开源的 Linux 发行版，感谢开源的强大力量，我们可以运行很多好玩的程序在上面。比如：科学上网，内网穿透，广告过滤，KMS 激活服务器，开启 VPN 通道......

## 正式开始折腾

### 第一步 ☝️：选择路由器

现在并不是所有的路由器都是支持 OpenWrt 的，可以在 OpenWrt 的官网查看是否支持你的机器[OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

当然也有的路由器原生就是 OpenWrt 系统，比如下面这个：

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Screen%20Shot%202023-04-14%20at%2017.18.07.png" alt="Screen Shot 2023-04-14 at 17.18.07" />

这样一个路由器也只支持 wifi5，有点太贵了（流下了贫穷的眼泪），为了追求极致的性价比（省钱），我决定自己找一款支持 OpenWrt 的路由器，同时也不是很贵

于是我找到了下面这款 👇

![Screen Shot 2023-04-14 at 17.24.04](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Screen%20Shot%202023-04-14%20at%2017.24.04.png)

只要 165 的价格！128MB 超大内存（对路由器来说确实很大）！支持 OpenWrt！年轻人的第一台路由器！

### 第二步 ✌️：如何安装 OpenWrt 呢

一般情况下来说，路由器本身的固件是不会给你一个接口把它换掉了，~~难道要用烧录器直接对着芯片烧录~~

但是事情在 Redmi AC2100 上还是有点希望的，Redmi AC2100 这个路由器有些版本固件出现了 bug（~~可能是故意的~~）我们可以通过一个 http 请求来开启固件的 ssh 登陆（看来 redmi 官方的固件也是基于 linux 的）
下面是具体步骤：

1. 刷入制定版本固件：首先我们要检查一下当前的固件版本有没有 bug，这边我建议直接刷入 2.0.23 版本的固件，我们使用[小米路由修复工具](http://bigota.miwifi.com/xiaoqiang/tools/MIWIFIRepairTool.x86.zip) 更新/降级固件版本至 [2.0.23](http://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/rm2100/miwifi_rm2100_all_fb720_2.0.23.bin)。

2. 查看你的**STOK**：使用浏览器打开你的路由登录界面，登录。复制地址栏的地址，我们需要地址里面的 stok 参数。参数如下所示，下文中的所有 **[STOK]** 字符串均指代这一 stok 参数，**[router-ip]**均代表路由器的地址，一般是 192.168.31.1

```URL
http://[router-ip]/cgi-bin/luci/;stok=[STOK]/web/home#router
```

3. 启用 SSH 服务器：复制下面的这一行地址到浏览器的地址栏，替换 stok 参数并访问。

```URL
http://[router-ip]/cgi-bin/luci/;stok=[STOK]/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%3B%20nvram%20set%20ssh_en%3D1%3B%20nvram%20commit%3B%20sed%20-i%20's%2Fchannel%3D.*%2Fchannel%3D%5C%22debug%5C%22%2Fg'%20%2Fetc%2Finit.d%2Fdropbear%3B%20%2Fetc%2Finit.d%2Fdropbear%20start%3B
```

4. 修改 root 密码：复制下面的这一行地址到浏览器的地址栏，替换 stok 参数并访问。

```URL
http://[router-ip]/cgi-bin/luci/;stok=[STOK]/api/misystem/set_config_iotdev?bssid=gallifrey&user_id=doctor&ssid=-h%0Aecho%20-e%20%27password%5Cnpassword%27%20%7C%20passwd%20root%0A
```

此命令执行完毕后，会将你的路由器的 root 用户密码修改为 `password`

执行完毕后，你已经可以通过 ssh 访问路由器了。既然我们可以 ssh 到路由器而且有 root 权限，那离成功只有一步之遥

5. 选择你想要刷入的固件：

- 官方固件[Index of /releases/22.03.4/targets/ramips/mt7621/ (openwrt.org)](https://downloads.openwrt.org/releases/22.03.4/targets/ramips/mt7621/)找到下面这个文件

  - xiaomi_redmi-router-ac2100-initramfs-kernel.bin
  - xiaomi_redmi-router-ac2100-squashfs-kernel1.bin
  - xiaomi_redmi-router-ac2100-squashfs-rootfs0.bin
  - xiaomi_redmi-router-ac2100-squashfs-sysupgrade.bin

- 第三方固件，可以自己在网上搜索，第三方的固件在官方的固件上安装了一些软件，就不用自己到处搜集资源安装了，功能相比于官方的更加强大，这里可以推荐一个[【2022-7-15】每日更新 稳定版/精简版 红米 小米 AC2100 Openwrt 固件 (提供定制)-小米无线路由器以及小米无线相关的设备-恩山无线论坛 - Powered by Discuz! (right.com.cn)](https://www.right.com.cn/forum/thread-7807773-1-1.html)

6. 正式刷入固件，这里以官方固件为例。

首先下载后缀为`squashfs-kernel1.bin`和`squashfs-rootfs0.bin`的两个文件，放入路由器中。然后使用下面的命令安装固件：

```SHELL
$ mtd write openwrt-ramips-mt7621-xiaomi_redmi-router-ac2100-squashfs-kernel1.bin kernel1
$ mtd -r write openwrt-ramips-mt7621-xiaomi_redmi-router-ac2100-squashfs-rootfs0.bin rootfs0
```

执行完毕后，系统将会自动重启进入 OpenWrt。

到这里为止，OpenWrt 就安装完成了，是不是很简单！

## 校园网的烦恼

总所周知，华科的校园网是要认证的，当我们兴冲冲地买了路由器，折腾了半天终于安装好 openwrt，发现根本没网！

别急！

有一个暂时的解决方案，直接连上路由器的网，打开http://172.18.18.60:8080/，直接在浏览器中像平时连校园网一样验证就行

有没有永久的解决方案呢，每次路由器重启了都要重新连也太麻烦了，特别是学校晚上要断电，那基本每天都要重新连，实在是太麻烦！

可以想到的是，有没有一个程序帮我们连校园网再加上开机自启动或者定时任务之类的就可以彻底解放了！

相信我们肯定不是第一个想到这个问题的人，我们的前辈已经在十年前想到了这个问题，于是 mentohust 就诞生了。但是别高兴太早，mentohust 已经很久不更新了，而且 mentohust 可以在 linux 上运行，没错，但是我们是在一个路由器上跑的 openwrt，硬件资源十分的匮乏，只有 128MB+128MB 的规格，没错，储存也只有 128MB！最关键的是这个路由器的**CPU 框架不是 x86 也不是 arm**，所以我们没法直接在 openwrt 上运行 mentohust。

后来我又在网上找了很久，终于找到了 mentohust 的替代品**minieap**!，而且最重要的我找到了 minieap 的 openwrt 的部署版本，因为 openwrt 是为了嵌入式设备开发的 linux 版本，跑在各种不同架构的 CPU 上，所以大部分软件都不会给直接运行的版本，而是需要从源码开始编译，但是路由器上连 build-essential 这种包都安装不了，更别提编译，所以我们要用到**交叉编译**，在我们正常的 linux 机器上编译路由器上能跑的程序。

openwrt 官方提供了交叉编译所需要的 sdkhttps://downloads.openwrt.org/releases/22.03.4/targets/ramips/mt7621/openwrt-sdk-22.03.4-ramips-mt7621_gcc-11.2.0_musl.Linux-x86_64.tar.xz，同时我们也找到了openwrt的专用的minieap版本[ysc3839/openwrt-minieap (github.com)](https://github.com/ysc3839/openwrt-minieap)，大功告成，下面是具体的步骤

1. 在 openwrt 下载对应 CPU 架构的 SDK 交叉编译包，如 redmi AC2100 是这个https://downloads.openwrt.org/releases/22.03.4/targets/ramips/mt7621/openwrt-sdk-22.03.4-ramips-mt7621_gcc-11.2.0_musl.Linux-x86_64.tar.xz
2. 到[ysc3839/openwrt-minieap (github.com)](https://github.com/ysc3839/openwrt-minieap)这个仓库下面，按照 README 的步骤，完成交叉编译
3. 生成可执行文件，将其上传到路由器上

校园网用的是锐捷验证，我的 minieap 执行参数如下：

```shell
$ minieap -u [studentid] -p [password] -n eth1 --module rjv3 -a 1 -d 2 -b 3
```

## 后记

1. 路由器在插网线的时候记得插的是 LAN 口还是 WAN 口。
2. 在用路由器科学上网后，可能会出现双层代理的问题，但是可以提供一个无感的科学上网体验，比如 switch 就可以直连外网。
3. 路由器的端口转发功能上十分实用，整个校园网就是一个大内网，将寝室的机器转发到校园网内网后就可以在校园任何地方连接寝室的服务器或者电脑。
4. 硬路由的性能还是有限的，而且可能会变成砖头 🧱，建议直接折腾软路由

参考链接 🔗

[[OpenWrt Wiki\] Welcome to the OpenWrt Project](https://openwrt.org/)

[红米 AC2100 路由器刷 OpenWrt 教程 – LotLab](https://www.lotlab.org/2021/06/13/install-openwrt-on-redmi-ac2100/)

[【2022-7-15】每日更新 稳定版/精简版 红米 小米 AC2100 Openwrt 固件 (提供定制)-小米无线路由器以及小米无线相关的设备-恩山无线论坛 - Powered by Discuz! (right.com.cn)](https://www.right.com.cn/forum/thread-7807773-1-1.html)

[ysc3839/openwrt-minieap (github.com)](https://github.com/ysc3839/openwrt-minieap)
