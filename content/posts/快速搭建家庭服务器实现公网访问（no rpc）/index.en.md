---
title: "Quick Setup of Home Server System with CasaOS"
date: 2024-05-01
draft: false
tags: ["NAS", "home-server", "casaos"]
categories: ["Tech", "DIY"]
keywords: ["casaos", "home server", "NAS system", "mini PC", "home cloud", "private cloud", "server setup", "self-hosted NAS"]
description: "Step-by-step tutorial: How to quickly set up a home server system using mini PC with CasaOS, creating personal NAS and private cloud storage solutions"
---

# Quick Setup Guide for CasaOS Home Server System
- [Quick Setup Guide for CasaOS Home Server System](#quick-setup-guide-for-casaos-home-server-system)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Installation Steps](#installation-steps)

## Introduction
CasaOS is a web operating system designed for home servers, allowing one-click installation of various software using Docker.  
http://demo.casaos.io/ (Official demo)  
https://github.com/IceWhaleTech/CasaOS (GitHub repository)  

## Prerequisites
1. A computer capable of running Linux - Recommended: Mini PC. I'm using an N100 chip model with 16GB RAM/1TB storage, planning to expand with another 1TB later. It dual-boots Windows (occupying 300GB), but Windows can be removed if desired.
2. A USB drive with ArchLinux installer - Required to install prerequisite software using yay (CasaOS script may not work directly with Arch, but should work with other distros).
3. Internet broadband with public IP
4. Accounts: Domain registrar (Alibaba Cloud, Tencent Cloud, etc.) and Cloudflare account. Cloudflare is needed because ISPs typically block ports 80 and 443, requiring workarounds.

## Installation Steps
1. Install Arch Linux (Skipped - many online tutorials available)
2. Install CasaOS
```shell
curl -fsSL https://get.casaos.io | sudo bash
```
For Arch Linux users: Install yay first (`sudo pacman -S yay`), then use yay to compile missing tools from source.

3. Domain Setup
- Why need domain: Public IP changes dynamically (with each dial-up connection or lease expiration), while domain remains constant. This requires Dynamic DNS (DDNS) technology.
- Install ddns-go on the host (recommended to install on router for better stability). If installed behind NAT, it relies on third-party APIs to detect public IP, which may be unstable.
- Purchase a domain (I used Tencent Cloud's .online domain), transfer DNS resolution to Cloudflare, add DNS A record (IP can be arbitrary initially), then obtain API token for DNS management.
- Configure ddns-go web interface (numerous online guides available).

4. NAT Configuration
- Assign static IP to Linux machine in router settings to prevent IP changes.
- Set up port forwarding: Forward Linux's port 80 to public port 81 (since port 80 is blocked by ISP). Other ports should maintain original port numbers.

5. Cloudflare Settings
- Add subdomain (e.g., home.xxxx.yyy) in Cloudflare. Create page rule to permanently redirect root domain (xxxx.yyy) to home.xxxx.yyy:81. Disable Cloudflare proxy for home.xxxx.yyy in DNS settings (Cloudflare proxy enables CDN caching only for specific ports like 80/443, which we don't use, and may slow down access due to overseas CDN nodes).
- For specific services requiring HTTPS (like some VSCode extensions): Create subdomain (e.g., im.xxxx.yyy), set port rule to forward requests to application port (e.g., 9876). This maintains clean domain in browser while accessing non-standard ports.
- For HTTPS: Use nginx reverse proxy with free Let's Encrypt certificates (renewable every 3 months using automation).

---
**2024-05-01 Update**
This guide focuses on home server deployment strategies without reverse proxy for public access (fixed access URLs).  
The port 80 blockage issue was initially mistaken for local firewall problems...  
This is a rough guide - feel free to leave comments if you need clarification.

---
**2024-05-28 Update**
CasaOS requires additional dependencies (e.g., auto USB mounting tools). Arch Linux users may encounter installation failures requiring manual dependency installation via yay (yay compiles from source with broader software availability than pacman).  
Recommended to use CasaOS officially supported systems for smoother installation.

20240528 Additional scattered notes:
1. **Hardware**: N100 mini PC (barebone), paired with off-brand 16GB RAM + 1TB SSD from PDD, plus 4TB HDD + enclosure from Taobao. Total cost ~1000 RMB (excluding HDD) - much cheaper than cloud server rental!
2. **Broadband**: Using China Telecom 100M home broadband. Important note: 100M refers to download speed - upload speed is crucial for servers. Compare upload speeds among ISPs for same download tier. Must request public IP and set modem to bridge mode (not router mode), then connect external router (recommend soft router). Annual cost 360 RMB (effectively free since home internet is needed anyway). Key point: Home broadband blocks ports 80/443 by default, so services can't use standard ports (though workarounds exist).
3. **Router config**: Assign static LAN IP to server. For services running on port 80, set up NAT forwarding 80:81 (external port 81). Use DDNS - some routers have built-in support. Recommend purchasing domestic domain + Cloudflare DNS hosting. For unsupported routers, install ddns-go on server. Search "ddns cloudflare" for detailed guides.

PS: Cloudflare helps bypass port 80/443 restrictions through various rules. No ICP备案 required as long as actual ports 80/443 aren't used.

---
2025-06-02 Update

Forgot how to mount Alist to Linux filesystem - adding notes here for reference.
After investigating running processes, I'm using rclone with custom configuration and systemd service.

```
[Unit]
Description=Alist (rclone)
After=network.target

[Service]
Type=notify
# Enable VFS mode, mount alist to /DATA/alist
ExecStart=/usr/bin/rclone mount alist:/ /DATA/alist --cache-dir /home/gyf/tmp --vfs-cache-max-size 30G --allow-other --vfs-cache-mode full --vfs-cache-max-age 24h0m0s --header "Referer:https://www.aliyundrive.com/drive" --timeout 2m30s --vfs-read-chunk-size-limit 2G --vfs-read-chunk-size 256M --no-checksum --vfs-read-ahead 256M --vfs-read-wait 20ms --transfers 8 --buffer-size 128M --dir-cache-time 60m --no-modtime --vfs-write-back 40m --umask 000
# Unmount
ExecStop=/usr/bin/fusermount -zu /DATA/alist
User=gyf
Restart=on-abort

[Install]
WantedBy=default.target
```