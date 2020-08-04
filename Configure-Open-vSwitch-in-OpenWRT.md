---
title: Configure Open vSwitch in OpenWRT
date: 2015-03-18 17:04:00
categories: Network
tags:
  - OpenWRT
  - Open vSwitch
aliases:
  - 2015/03/18/Configure-Open-vSwitch-in-OpenWRT/
---
當你安裝完 Open vSwitch 或是燒了一個[內建 Open vSwitch 的 OpenWRT](http://worldend.logdown.com/posts/256561-compile-openwrt-with-open-vswitch) 後，接下來要將網卡綁到 Open vSwitch 上。

# **實驗環境**

- 路由器: [TP-Link TL-WR1043ND](http://www.tp-link.tw/products/details/?model=TL-WR1043ND)
    - Wan port x 1
    - Lan port x 4

<!-- more -->
----

# Prepare Work

- **更換 root 密碼**
    剛裝好 OpenWRT 之後，要先設定 root 密碼。第一次登入必須使用 telnet 從 lan 登入:
```
telnet 192.168.1.1
```
    接下來執行 `passwd` 更換 root 密碼:
```
root@OpenWrt:/# passwd
Changing password for root
New password:
Retype password:
Password for root changed by root
```

    執行 `exit` 登出，然後改用 ssh 登入。 (或者是 Luci)
```
ssh root@192.168.1.1
```

## Modify network configuration

Network 設定檔在 `/etc/config/network`，先看一下 **lan** 預設值:

```python /etc/config/network
config interface 'lan'
	option ifname 'eth1'
	option force_link '1'
	option type 'bridge'
	option proto 'static'
	option ipaddr '192.168.1.1'
	option netmask '255.255.255.0'
	option ip6assign '60'

```

- `lan` 是**網路名稱** 而非網卡名稱
- ifname 後面接網卡名稱，`eth1` (lan port)。**每一個 network 只能設定一個 ifname**。
- proto 後面接設定網路的方式: 固定 IP 就是 `static`、浮動 IP 就是 `dhcp`；其他還有 ppp ...等。

lan 的 type 為 `bridge`，使用 `ifconfig` 與 `brctl` 可以觀察到:

```
root@OpenWrt:~# ifconfig br-lan
br-lan    Link encap:Ethernet  HWaddr E8:DE:27:67:05:6E  
          inet addr:192.168.1.1  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fddb:c62a:a451::1/60 Scope:Global
          inet6 addr: fe80::eade:27ff:fe67:56e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:48715 errors:0 dropped:0 overruns:0 frame:0
          TX packets:51387 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:9282220 (8.8 MiB)  TX bytes:22974165 (21.9 MiB)

root@OpenWrt:~# brctl show
bridge name	bridge id		STP enabled	interfaces
br-lan		7fff.e8de2767056e	no		eth1
```


接下來我們要做的事情是:

1. 設定 lan 的網卡為 Open vSwitch 的 switch name
2. 設定新的 network 來開啟原本的網卡

因此，第一步就是要修改 `/etc/config/network` :

```python /etc/config/network
config interface 'lan'                         
        option ifname 'ovs-lan'                
        option proto 'static'                  
        option ipaddr '192.168.1.1'            
        option netmask '255.255.255.0'         

config interface 'eth1'               
        option ifname 'eth1'          
        option proto 'static'
```

接下來，要使用指令來做幾個步驟:

- 建立 ovs bridge
- 移除 linux bridge 上原本的 interface
- 將 interface 新增到 ovs bridge 中
- 重新啟動網路套用新的設定值

以上步驟可以利用 script 一氣呵成。 (當然也可以手動慢慢打指令，但是要注意的是執行 `brctl delif` 與 `ovs-vsctl add-port` 時會造成網路斷線)

新增一個 scrpit:

```sh ovs.sh
#!/bin/sh
OVS_LAN="ovs-lan"
LAN_PORT="eth1"
LINUX_BRIDGE="br-lan"

# Create Open vSwitch
ovs-vsctl --may-exist add-br $OVS_LAN

# Remove LAN port from Linux bridge
brctl delif $LINUX_BRIDGE $LAN_PORT

# Add LAN port to Open vSwitch
ovs-vsctl --may-exist add-port $OVS_LAN $LAN_PORT

# Restart network
/etc/init.d/network restart

exit 0
```

然後執行這個 script
```
sh ovs.sh &
```

這樣就完成了 Open vSwitch 的基礎設定。

```
root@OpenWrt:~# ovs-vsctl show
f9a6117b-8af2-400b-91ff-39986349f0c6
    Bridge ovs-lan
        Port ovs-lan
            Interface ovs-lan
                type: internal
        Port "eth1"
            Interface "eth1"
```

## Wifi

Wifi 的設定檔放在 `/etc/config/wireless`

```text /etc/config/wireless
onfig wifi-device  radio0
        option type     mac80211
        option channel  11
        option hwmode   11g
        option path     'platform/qca955x_wmac'
        option htmode   HT20
        # REMOVE THIS LINE TO ENABLE WIFI:
        option disabled 1

config wifi-iface
        option device   radio0
        option network  lan
        option mode     ap
        option ssid     OpenWrt
        option encryption none
```

註解 `option disabled 1` 這行，然後執行 `wifi`

執行 `ifconfig` 可以看到 `wlan0` 這個介面

```
wlan0     Link encap:Ethernet  HWaddr E8:DE:27:67:05:6E  
          inet6 addr: fe80::eade:27ff:fe67:56e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:480 (480.0 B)
```

但是利用 ovs-vsctl show 卻看到 wlan0 沒有被新增到 openvswitch 裡:

```
root@OpenWrt:~# ovs-vsctl show
f9a6117b-8af2-400b-91ff-39986349f0c6
    Bridge ovs-lan
        Port ovs-lan
            Interface ovs-lan
                type: internal
        Port "eth1"
            Interface "eth1"
```

手動執行 `ovs-vsctl add-port` 去新增 wlan0
```
ovs-vsctl add-port ovs-lan wlan0
```

這時候再用 ovs-vsctl show 就會看到 wlan0 加入到 openvswitch 中。使用無線裝置測試連線，可以順利取得 IP 並連上 internet。

# Reference

- [OpenvSwitch Lab 7$ Setting OpenWrt](http://roan.logdown.com/posts/239799-openvswitch-lab-7-setting-openwrt)
- [OpenFlow 1.3 for OpenWRT on TL-1043ND with OVS](http://linton.tw/2014/05/13/openflow-13-for-openwrt-on-tl-1043nd-with-open-vswitch/)

----

# 附錄

如果想要自己手動打指令的方式去設定，有兩種方法:

1. 參考 [OpenvSwitch Lab 7$ Setting OpenWrt](http://roan.logdown.com/posts/239799-openvswitch-lab-7-setting-openwrt) 這篇的做法，切出一個 console port

2. 或者是你跟我一樣懶XD，那就修改 firewall，然後從 wan 登入進去設定。

編輯 `/etc/config/firewall` 並加上以下內容:

```
config rule
        option src              wan
        option dest_port        22
        option target           ACCEPT
        option proto            tcp
```

然後重啟 firewall

```
/etc/init.d/firewall restart
```

之後就可以從 wan 登入 OpenWRT 了

Firewall 的規則可以參考 OpenWRT 官方的資料: [Firewall configuration](http://wiki.openwrt.org/doc/uci/firewall)
