---
title: IPv6 Neighbour table overflow
date: 2015-05-01 12:07:00
categories: Network
tags:
  - GRUB
  - IPv6
  - OpenStack
  - Ubuntu
aliases:
  - 2015/05/01/IPv6-Neighbour-table-overflow/
---
這兩天一直在 syslog 上看到 `Neighbour table overflow`

```syslog /var/log/syslog
Apr 30 06:27:03 host-42 kernel: [72924.290265] net_ratelimit: 1739 callbacks suppressed
Apr 30 06:27:03 host-42 kernel: [72924.290269] IPv6: Neighbour table overflow
```

每兩分鐘就出現6~7筆,數量非常的多
如果不處理他，放任它繼續增長，過一兩天後系統就會 kernel panic。

<!-- more -->

Google 一番之後大概得知，neighbour table 基本上就是 arp table。所以簡單來說就是 arp table 爆了。
問題是我已經把 IPv6 都關了，還能衝爆 neighbour table 實在無法理解。

## Increase the size of neighbour table

這大概是搜尋到最多的解法，簡單來說就是直接增加 neighbour table 的大小。

設定 `/etc/sysctl.conf` :
```python /etc/sysctl.conf
net.ipv6.neigh.default.gc_thresh1 = 4096
net.ipv6.neigh.default.gc_thresh2 = 8192
net.ipv6.neigh.default.gc_thresh3 = 8192
```

然後在執行 `sysctl -p` 讓設定值生效。

在網路上搜尋到的範例大多都是設定成 512, 1024, 2048。這些設定對我的平台一點效果都沒有，必須要設定到 4096, 8192 才夠用。

## Disable IPv6

原本我是在 `/etc/sysctl.conf` 中加入以下設定來關閉 IPv6：

```python /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

但是很顯然的，這並沒有真正的關閉 IPv6。到比較像是單純隱藏 IPv6 的感覺。最好的證明就是：

1. `netstat -tnlup` 可以看到部分 service 依然監聽 IPv6
2. Neighbour table overflow 的狀況出現

因此我嘗試了一下用 grub 的方式關閉，沒想到效果不錯。

1. 編輯 `/etc/default/grub`, 並且在 `GRUB_CMDLINE_LINUX` 的設定中加入 `ipv6.disable=1`
```bash /etc/default/grub
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```
2. 更新 grub `update-grub`
3. Reboot

設定完之後發現:

- `netstat -tnlup` 再也看不到有 service listen on IPv6 address
- 先前加的 sysctl 參數，不論是 disable IPv6 還是 increase neighbour table size 都直接失效。
```
root@localhost:~ # sysctl -p
error: "net.ipv6.conf.all.disable_ipv6" is an unknown key
error: "net.ipv6.conf.default.disable_ipv6" is an unknown key
error: "net.ipv6.conf.lo.disable_ipv6" is an unknown key
error: "net.ipv6.neigh.default.gc_thresh1" is an unknown key
error: "net.ipv6.neigh.default.gc_thresh2" is an unknown key
error: "net.ipv6.neigh.default.gc_thresh3" is an unknown key
```
  即使將 disable\_ipv6 都設定為 0 ，也無法啟用 IPv6。


## 現況與採取方案

一開始先採用了第一個方案，但是數值設太小了，沒多久系統又爆出一樣的 error；於是採取的第二個方案，在 grub 的參數上增加 `ipv6.disable=1`。結果 syslog 終於沒有在看到這個 error，但是 syslog 中出現另外一個錯誤：

```
Apr 28 06:50:52 host-42 ovs-vsctl: 00001|vsctl|INFO|Called as /usr/bin/ovs-vsctl --timeout=2 add-port br-tun gre-10
Apr 28 06:50:52 host-42 ovs-vsctl: 00002|vsctl|ERR|cannot create a port named gre-10 because a port named gre-10 already exists on bridge br-tun
Apr 28 06:50:53 host-42 ovs-vsctl: 00001|vsctl|INFO|Called as /usr/bin/ovs-vsctl --timeout=2 set Interface gre-10 type=gre
Apr 28 06:50:53 host-42 ovs-vsctl: 00001|vsctl|INFO|Called as /usr/bin/ovs-vsctl --timeout=2 set Interface gre-10 options:remote_ip=192.168.91.93
Apr 28 06:50:53 host-42 ovs-vsctl: 00001|vsctl|INFO|Called as /usr/bin/ovs-vsctl --timeout=2 set Interface gre-10 options:in_key=flow
Apr 28 06:50:53 host-42 ovs-vsctl: 00001|vsctl|INFO|Called as /usr/bin/ovs-vsctl --timeout=2 set Interface gre-10 options:out_key=flow
```

這個問題很久以前就看過，但是以前都不影響 OpenStack 的運作；而這次 syslog 中每一秒都出現這個 error，OpenStack virtual network 也跟著一起掛了。嘗試將 IPv6 打開，這個 error 就消除了。這讓我感到非常莫名其妙，目前也還不太清楚發生的原因。

最後還是回到第一個方案，並起將參數調大至 4096 與 8192，最後才讓系統恢復正常運作。但我心中認為這是個治標不治本的方案。看來只能多花點時間研究一下 IPv6 的機制了。


## Reference

- [Neighbour table overflow – debug – IPv4 and IPv6](http://www.arcweb.ro/blog/2011/12/13/neighbour-table-overflow-debug-ipv4-and-ipv6/)
- [neighbour table overflow 问题解决](http://blog.csdn.net/reyleon/article/details/24981581)**(Important)**
