---
title: "快速搭建家庭服务器系统casaos"
date: 2024-05-01
draft: false
tags: ["NAS", "家庭服务器", "casaos"]
categories: ["技术", "折腾"]
keywords: ["casaos", "家庭服务器", "NAS系统", "迷你主机", "家庭云", "私有云", "服务器搭建", "自建NAS"]
description: "详细教程：如何使用迷你主机快速搭建基于casaos的家庭服务器系统，打造个人NAS和私有云存储解决方案"
---

# 快速搭建家庭服务器系统casaos
- [快速搭建家庭服务器系统casaos](#快速搭建家庭服务器系统casaos)
  - [简介](#简介)
  - [提前准备](#提前准备)
  - [正式安装](#正式安装)

## 简介
casaos是一款致力于家庭服务器的网络操作系统，使用docker一键式安装各种软件  
http://demo.casaos.io/ 这是人家的demo  
https://github.com/IceWhaleTech/CasaOS github开源  

## 提前准备
1. 一台可运行linux的电脑--这边我推荐迷你主机，我用的是芯片N100，16G/1T的，打算将来再扩展1T。主要装了双系统，windows占了300G，也可以尝试着去除windows（没必要windows）
2. 一个U盘，archLinux的安装盘--> 需要用yay安装一些前置软件（casa的脚本没法一键安装，不用arch应该可以一键安装）
3. 一个有公网ip的宽带
4. 中国的域名购买网站的账号（阿里云、腾讯云等）和cloudflare账号： 为什么这里需要cloudfare，就是第三点有个很坑的地方，现在的服务商会把公网的80和443端口禁掉，我们需要用些方法绕过去。

## 正式安装
1. 安装arch liunx 跳过~~网上很多教程
2. 安装casaos
```shell
curl -fsSL https://get.casaos.io | sudo bash
```
中间会有很多问题，如果你安装的linux也是arch的话,需要安装yay, sudo pacman -S yay,然后使用yay从源码层面安装缺少的工具  

3. 域名  
- 需要域名的原因：我们分配到的公网IP是会变的，每次拨号上网就会改变，租赁ip时间到期可能会变，但是域名是不会变的；关键是怎么把动态的ip绑定到静态的域名上  
这就需要动态dns的技术了---ddns
- 我是在主机下安装了ddns-go,其实推荐在软路由上安装，因为可以通过读取软路由的网卡的ip地址（就是公网）来给ddns服务器发送ip；如果用在NAT下的主机做ddns客户端的话，只能依赖第三方接口获取自己的公网ip，就产生了依赖，具有不稳定性  
- 我是在腾讯云买了online域名，dns解析转移到cloudflare下，添加好dns的A解析记录，注意ip可以任意填写；然后获取能操作dns解析权限的token，ddns-go需要
- 打开ddns-go的web页面填写好，总之这一步也可以在网络上找怎么做，我不赘述了

4. NAT设置  
- 在路由器（最好是软路由）找到linux机子的ip绑定为静态，防止每次连接或者ip到期后发生改变   
- NAT转换那块，将linux的80端口指向公网的81端口，用81是因为外网80端口是被服务商禁止掉了；其他端口转换应该保持原端口号   
5. cloudflare设置
- cloudflare在多设置一个域名，比如home.xxxx.yyy，再设置一个页面规则，将原来的根域名xxxx.yyy永久重定向到home.xxxx.yyy:81，同时记得把home.xxxx.yyy在域名解析那边关掉cloudflare代理。 解释下cloudflare代理是开启cdn的，只会去缓存特定端口如80、443，而我们的机子这些端口是不开放的，并且它的cdn主要在国外，基本只会减速。
- 某些特殊的网站，想要安全性或者可用性（vscode一些插件必须要https），可以使用cloudflare的cdn。可以设置一个子域名im.xxxx.yyy,设置端口规则，将主机名为im.xxxx.yyy的请求转发到你应用所在端口，比如9876。这样游览器中的域名就永远保持im.xxxx.yyy实际上访问的又是主机的9876端口了
- 关于https，建议就全部走反向代理nginx，证书可以向Let's Encrypt申请，免费；三个月到期后继续申请(使用自动化程序到期前一个月重新申请)，永久免费的。

---
**2024-05-01 补充**
主要是提供一些部署家庭服务器的思路，不通过反向代理公网访问（固定访问链接）  
特别是80端口被禁，一直以为是我本地防火墙还是啥问题。。。。  
然后很粗略，如果有人搜到，不懂可以留言，我再完善；  

---
**2024-05-28 补充**
casaos本身是需要依赖一些其他的软件的，比如自动挂载USB设备的程序；这些如果是arch linux中执行安装的shell脚本可能会出现安装失败，需要手动安装依赖；可以使用yay手动安装（源自己处理好，不要用pacman有些软件是没有的；pacman和yay的区别在于yay是从源码安装的，然后软件相对会多些）。
当然，最好使用casaos官方支持的系统，我想应该会好安装些  

20240528 这边加些步骤，零散记录  
1. **硬件**我是N100小主机，买的时候买不带内存和硬盘的，然后再pdd上便宜买些杂牌的16G内存+1T固态硬盘，最后淘宝买了4T机械硬盘+硬盘盒子。除开机械硬盘外一共正好花了1000RMB！这个价格可比租服务器便宜得多得多！
2. **宽带**使用电信的100M的家庭宽带，这里有个坑，100M是下行宽带，做服务器重要的是上行宽带；所以建议对比电信、联通、移动的同一下行速率的宽带的上行速率。记住，一定要说明你需要公网IP，并且光猫不要调成路由模式，只要ap接入模式，自己外接一个路由器（我建议可以自己搞个软路由网关）。这个费用一年360，但是本来家里就需要宽带的，所以等于没花钱。重点，家庭宽带是默认不会开启80http、443https这种关键端口的,所以软件对外不能是默认端口（实际上也有办法不输入端口）
3. **配置路由**，服务器开启后，一定要在路由器中固定好局域网IP。然后运行服务软件，比如服务运行在80端口，那么可以在路由器上端口映射NAT设置80:81，映射到外网是81端口。然后就是用DDNS技术，有些路由器是有ddns的，建议在国内买域名然后给cloudflare托管。如果路由器不支持（用软路由不可能不支持，普通路由可能不支持cloudflare），那么就在服务器上搭建一个ddns-go服务（我是用这个服务的，ddns服务有很多，可以随便选个自己认为好的）。然后就百度、Google，关键字“ddns cloudflare”。哇哈哈，此处就省略了

ps. 使用cloundflare的原因，主要是可以通过各种规则绕开80、443封锁的问题；
不论用什么dns服务商，以上方法只要不实际用到80、443，都可以不用备案

---
2025-06-02 补充

我忘了alist如何挂载到linux文件系统上，本来想着这边有记录的。发现没有，故此补充记录。
在服务器上经过查证所有活动进程后，我使用的时rlone，然后自定义配置，启动用户systemd服务。
因为现在有deepseek这些LLM，直接查rclone使用即可，此处只做记录。

```
[Unit]
Description=Alist (rclone)
# Make sure we have network enabled
After=network.target

[Service]
Type=notify

# 启用vfs模式，将alist挂载给/DATA/alis:文件夹
ExecStart=/usr/bin/rclone mount alist:/ /DATA/alist --cache-dir /home/gyf/tmp --vfs-cache-max-size 30G --allow-other --vfs-cache-mode full --vfs-cache-max-age 24h0m0s --header "Referer:https://www.aliyundrive.com/drive" --timeout 2m30s --vfs-read-chunk-size-limit 2G --vfs-read-chunk-size 256M --no-checksum --vfs-read-ahead 256M --vfs-read-wait 20ms --transfers 8 --buffer-size 128M --dir-cache-time 60m --no-modtime --vfs-write-back 40m --umask 000
# 取消挂载
ExecStop=/usr/bin/fusermount -zu /DATA/alist

User=gyf

# Restart the service whenever rclone exists with non-zero exit code
Restart=on-abort

[Install]
# Autostart after reboot
WantedBy=default.target
```