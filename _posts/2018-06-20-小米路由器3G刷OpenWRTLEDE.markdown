---
title: "小米路由器 3G 刷 OpenWRT/LEDE"
date: 2018-06-20 11:49:29
categories: ["2018"]
tags: [openwrt]
---

2018.6月某宝入手 [小米路由器 3G](https://www.mi.com/miwifi3g/) 价格 179 元，结合配置来看性价比高，适合用来折腾，遂刷机。

配置信息如下：

|           | 参数                                             |
| :-------- | ------------------------------------------------ |
| 处理器    | MT7621A MIPS双核880MHz                           |
| ROM       | 128MB SLC Nand Flash                             |
| 内存      | 256MB DDR3-1200                                  |
| 2.4G WiFi | 2X2（支持IEEE 802.11N协议，最高速率可达300Mbps） |
| 5G WiFi   | 2X2（支持IEEE 802.11AC协议，最高速率可达867Mbps  |
| USB       | 3.0                                              |
| LAN/WAN   | 3个10/100/1000M自适应                            |

## 准备工作

+ 网线（虽然不是必须，但这可以节省很多工作）
+ 一台电脑，Windows 需安装 ssh 工具（putty，winscp），linux 自带 ssh
+ 一个 4G 以上 U盘（格式化为 FAT）
+ 注册小米帐号 <https://account.xiaomi.com/>
+ 下载固件 <http://www.miwifi.com/miwifi_download.html> 选择 **ROM for R3G 开发版**
+ 下载小米 **miwifi_ssh_bin** 文件 <https://d.miwifi.com/rom/ssh> 下载工具包会提示放弃保修服务，无视它，记录下页面上的 root 密码，用于 ssh 连接
+ 下载 openwrt snapshot 文件

    - [openwrt-ramips-mt7621-mir3g-squashfs-kernel1.bin](https://downloads.openwrt.org/snapshots/targets/ramips/mt7621/openwrt-ramips-mt7621-mir3g-squashfs-kernel1.bin)
    - [openwrt-ramips-mt7621-mir3g-squashfs-rootfs0.bin](https://downloads.openwrt.org/snapshots/targets/ramips/mt7621/openwrt-ramips-mt7621-mir3g-squashfs-rootfs0.bin)

## 安装步骤

1. 网线连接电脑与路由器 LAN 口，打开浏览器，登录路由器管理界面 <http://192.168.31.1>，选择升级系统，将下载的固件 **ROM for R3G 开发版**上传，确认等路由器更新完毕。

2. 将 **miwifi_ssh.bin** 文件复制到 U盘中。

3. 关闭路由器电源，将U盘插入到路由器，用回形针插入 reset 按钮，开启电源，大约 10 秒钟，路由器前方指示灯黄色闪烁表示在安装 ssh 文件，松开回形针。期间路由器会重启，直到指示灯变蓝。

4. 现在可以登录到路由器了，`ssh root@192.168.1.1`

    ![ARE U OK](/images/openwrt.png)

5. 将两个 openwrt 文件 scp 到小米路由器，`scp openwrt-* 192.168.1.1:/tmp`

6. 开始刷机

    ```
    root@OpenWrt:~# mtd write openwrt-ramips-mt7621-mir3g-squashfs-kernel1.bin kernel1
    root@OpenWrt:~# mtd write openwrt-ramips-mt7621-mir3g-squashfs-rootfs0.bin rootfs0
    root@OpenWrt:~# nvram set flag_last_success=1
    root@OpenWrt:~# nvram commit
    root@OpenWrt:~# reboot
    ```

7. 等路由器刷机完毕，再 ssh 登录到路由器，修改 root 密码，以及电信拨号网络设置

    ```
    root@phantom-thinkpad:~# ssh root@192.168.1.1
    # 编辑 network 配置文件 设置 wan 口拨号上网
    root@OpenWrt:~# vi /etc/config/network
    config interface 'wan'
            option proto 'pppoe'
            option mtu '1480'
            option special '0'
            option username 'yourusername'
            option password 'yourpassword'
            option ifname 'eth1'
    # 重启网络
    root@OpenWrt:~# /etc/init.d/network restart
    # 安装 luci 管理界面
    root@OpenWrt:~# opkg update
    root@OpenWrt:~# opkg install uhttpd luci luci-app-uhttpd
    ```

8. 打开浏览器，登录 <http://192.168.1.1/> 进入 luci 页面配置路由器。

9. 至此刷机完毕，enjoy it！

## 参考链接

<https://enterpriseadmins.blogspot.com/2017/09/step-by-step-openwrtlede-base.html>

<https://klseet.com/267-lede/lede-miwifi/392-miwifi-3g-lede-unifi-ready>
