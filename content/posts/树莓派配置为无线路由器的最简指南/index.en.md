---
title: "Minimal Guide to Configure Raspberry Pi as a Wireless Router"
date: 2020-06-08
draft: false
tags: ["raspberrypi", "router", "linux", "network"]
categories: ["DIY"]
keywords: ["raspberry pi", "wireless router", "linux routing", "hostapd", "dnsmasq", "iptables", "network forwarding", "AP mode"]
description: "Step-by-step guide to configure Raspberry Pi as a wireless router with access point and network forwarding capabilities"
---

# Minimal Guide to Configure Raspberry Pi as a Wireless Router

## 0. Prerequisites

**Hardware Environment**: Raspberry Pi 3 B+ (compatible with other models)

**Software Environment**: [Raspberry Pi Official 32-bit OS](https://www.raspberrypi.org/downloads/raspberry-pi-os/), recommended to change software sources for faster downloads.

### Change Software Sources

```shell
sudo vim /etc/apt/sources.list
```

Comment out original sources and add Tsinghua sources:
```
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
```

```shell
sudo vim /etc/apt/sources.list.d/raspi.list
```
Add:
```
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

Update package list:
```shell
sudo apt update
```

## 1. Install Required Software

```shell
sudo apt install dnsmasq hostapd
```

- **dnsmasq**: Provides DNS and DHCP services
- **hostapd**: Configures wireless card as Access Point

## 2. Configure Network Interfaces

```shell
sudo vim /etc/network/interfaces
```

Add the following configuration at the beginning of the file:

```
# Ethernet interface configuration
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp

# Wireless interface configuration
allow-hotplug wlan0
iface wlan0 inet static
    address 192.168.2.1
    netmask 255.255.255.0
    network 192.168.2.0
    broadcast 192.168.2.255

# Restore iptables rules on startup
up iptables-restore < /etc/iptables.ipv4.nat
```

**Configuration Notes**:
- `eth0`: Ethernet interface, set to DHCP for automatic IP acquisition
- `wlan0`: Wireless interface, configured with static IP `192.168.2.1`
- After configuration, this wireless interface cannot connect to other WiFi networks
- Broadcast address: `192.168.2.255`, Network address: `192.168.2.0`, Subnet mask: `255.255.255.0`

## 3. Configure hostapd (Wireless Access Point)

```shell
sudo vim /etc/hostapd/hostapd.conf
```

Create new file if it doesn't exist, add the following content:

```ini
# Wireless interface name
interface=wlan0

# Use nl80211 driver
driver=nl80211

# Network name (SSID)
ssid=passwd-is-81

# Use 2.4GHz band
hw_mode=g

# Channel
channel=6

# Enable 802.11n
ieee80211n=1

# Enable WMM
wmm_enabled=1

# Enable 40MHz channels
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]

# MAC address access control
macaddr_acl=0

# Authentication algorithms
auth_algs=1

# Hide SSID setting
ignore_broadcast_ssid=0

# Use WPA2
wpa=2

# WPA key management
wpa_key_mgmt=WPA-PSK

# Network passphrase
wpa_passphrase=11111111

# Encryption method
rsn_pairwise=CCMP
```

**Hotspot Information**:
- SSID: `passwd-is-81`
- Password: `11111111`
- Band: 2.4GHz

## 4. Configure dnsmasq (DNS and DHCP)

```shell
sudo vim /etc/dnsmasq.conf
```

If the file has existing content, comment it out and add:

```ini
# Use wlan0 interface
interface=wlan0

# Listen address
listen-address=192.168.2.1

# Bind to interface
bind-interfaces

# DNS server
server=114.114.114.114

# Don't forward short names
domain-needed

# Never forward non-routed addresses
bogus-priv

# DHCP address range
dhcp-range=192.168.2.2,192.168.2.254,12h
```

**Important**: Ensure IP addresses match the network interface configuration from Step 2.

## 5. Configure iptables (Network Forwarding)

iptables is Linux's firewall tool used to configure network packet forwarding rules.

```shell
# Set up NAT forwarding
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Set up forwarding rules
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

# Save rules
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Saved rules will be automatically restored on system startup (via configuration in Step 2).

## 6. Enable IPv4 Forwarding

```shell
sudo vim /etc/sysctl.conf
```

Find and uncomment (or add):
```
net.ipv4.ip_forward=1
```

Or apply immediately:
```shell
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```

## 7. Configure Auto-start on Boot

Use systemctl to configure services to start automatically:

```shell
sudo systemctl enable dnsmasq
sudo systemctl enable hostapd
```

If you encounter error `Failed to start hostapd.service: Unit hostapd.service is masked.`, execute:

```shell
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

**Other Useful Commands**:
- Start services immediately: `sudo systemctl start dnsmasq` / `sudo systemctl start hostapd`
- Check service status: `sudo systemctl status dnsmasq` / `sudo systemctl status hostapd`
- Restart services: `sudo systemctl restart dnsmasq` / `sudo systemctl restart hostapd`

## Verification

Reboot Raspberry Pi after configuration:

```shell
sudo reboot
```

After reboot, search for WiFi network `passwd-is-81` on other devices, connect with password `11111111`, and you should be able to connect normally and access the internet.

## Troubleshooting

1. **Cannot connect to internet**: Check if eth0 obtained IP address normally
2. **Cannot obtain IP address**: Check if dnsmasq service is running properly
3. **Cannot find WiFi signal**: Check hostapd service status and configuration
4. **No internet after connection**: Check iptables rules and IP forwarding settings
```
