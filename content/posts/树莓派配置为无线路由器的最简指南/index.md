---
title: "树莓派配置为无线路由器的最简指南"
date: 2020-06-08
draft: false
tags: ["树莓派", "路由器", "Linux", "网络"]
categories: ["折腾"]
keywords: ["树莓派", "无线路由器", "Linux路由", "hostapd", "dnsmasq", "iptables", "网络转发", "AP模式"]
description: "详细指导如何在树莓派上配置无线路由器功能，实现无线接入点和网络转发"
---
## 树莓派在linux环境下配置成路由器的最简操作

0. 前置
1. 安装软件
2. 配置网卡
3. 配置hostapd
4. 配置dnsmasq
5. 配置iptables
6. 开启ipv4转发
7. 配置开机自启动

----

### 0.前置

硬件环境： 树莓派3 b+

软件环境为[树莓派官方系统32位](https://www.raspberrypi.org/downloads/raspberry-pi-os/)， 建议更换软件源。

```shell
sudo vim /etc/apt/sources.list
// 注释掉原来的源，添加新源
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
```

```shell
vim /etc/apt/sources.list.d/raspi.list
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

### 1. 安装软件

```shell
sudo apt install dnsmasq hostapd
```

前者为dns和DHCP服务器。后者可以使无线网卡成为无线接入点。

### 2.配置网卡

```
sudo vim /etc/network/interfaces
然后将下面这段插入第一条非注释语句的前面
----
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug wlan0
iface wlan0 inet static
    address 192.168.2.1
    netmask 255.255.255.0
    network 192.168.2.0
    broadcast 192.168.2.255

up iptables-restore < /etc/iptables.ipv4.nat
```

第一块配置的是有限网卡， 设置为自动连接。

第二部分是无线网卡，记住一旦这样配置后，接下来它就无法用于连接wifi了。我这边将其配置成192.168.2.1，忘了不与其他路由器重合。广播地址192.168.2.255，网络地址192.168.2.0，掩码/24，网关192.168.2.1

第三步部分是调用iptables的配置文件。该配置文件应该是会将两个网关进出ip包连成一个通道（第五步骤会做）。

### 3.配置hostapd

```
sudo vim /etc/hostapd/hostapd.conf
---
可能会没有这个文件，反正会新建
----
# This is the name of the WiFi interface we configured above
interface=wlan0

# Use the nl80211 driver with the brcmfmac driver
driver=nl80211

# This is the name of the network
ssid=passwd-is-81

# Use the 2.4GHz band
hw_mode=g

# Use channel 6
channel=6

# Enable 802.11n
ieee80211n=1

# Enable WMM
wmm_enabled=1

# Enable 40MHz channels with 20ns guard interval
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]

# Accept all MAC addresses
macaddr_acl=0

# Use WPA authentication
auth_algs=1

# Require clients to know the network name
ignore_broadcast_ssid=0

# Use WPA2
wpa=2

# Use a pre-shared key
wpa_key_mgmt=WPA-PSK

# The network passphrase
wpa_passphrase=11111111

# Use AES, instead of TKIP
rsn_pairwise=CCMP
```

热点名<code>passwd-is-81</code>, 密码<code>11111111</code>.其他方式和一般2.4ghz频率的信号一样.

### 4. 配置dnsmasq

```
sudo vim /etc/dnsmasq.conf
---
如果该文件存在东西了，可以注释掉
----

# Use interface wlan0
interface=wlan0

# Explicitly specify the address to listen on
listen-address=192.168.2.1

# Bind to the interface to make sure we aren't sending things elsewhere
bind-interfaces

# Forward DNS requests to Google DNS
server=114.114.114.114

# Don't forward short names
domain-needed

# Never forward addresses in the non-routed address spaces
bogus-priv

# Assign IP addresses between 192.168.2.2 and 192.168.2.254 with a 12 hour lease time
dhcp-range=192.168.2.2,192.168.2.254,12h

```

记住这边的IP地址，除了dns-server的，其他都要和网卡信息符合，也就是你之前配置的。

### 5. 配置iptables

介绍： 可以把iptables理解成是一个防火墙，其实本身就是防火墙。在做代理时也会用到它。

```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE  
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT  
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT 

```

以下部分是记录下当前iptables， 每次重启恢复一遍（恢复操作是在2.配置网卡最后一块配置里执行）

```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

### 6.开启ipv4转发

```
sudo vim /etc/sysctl.conf
----
将net.ipv4.ip_forward=1反注释掉，或者直接在最后输入
----
net.ipv4.ip_forward=1
```

或者直接以下操作

```shell
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```



### 7.配置开机自启动

使用 systemctl 工具配置 dnsmasq 和 hostapd 开机自启动。将其中的 enable 替换为 start 和 status ，可以实现立即启动软件和查看软件当前状态。

```
$ sudo systemctl enable dnsmasq
$ sudo systemctl enable hostapd
```

我在配置时出现错误信息 `Failed to start hostapd.service: Unit hostapd.service is masked. `使用下面命令可以解决。

```
$ sudo systemctl unmask hostapd
```

