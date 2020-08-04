---
title: OpenStack Nova cannot reboot instance - Unable to create tap device
date: 2015-02-11 16:36:00
categories: OpenStack
tags:
  - Nova
  - OpenStack
aliases:
  - 2015/02/11/OpenStack-Nova-cannot-reboot-instance-unable-to-create-tap-device/
---

早上因為某種因素，將 nova compute 強制重開機。當開機完成之後，使用 `nova reboot --hard <server>` 的方式，想要開啟instance 卻失敗，在 `/var/log/nova/nova-compute.log` 中看到以下錯誤訊息:

```
2015-02-11 16:10:54.110 ERROR nova.compute.manager [req-a3d9cf35-82ee-4857-b69d-99ef0c8ca753 b6a90e8c63ad4612917655fb9b04ad92 ecb687200c6a4574bdaf3ea3633c6b3f] [instance: 7bdad622-dd70-49d7-89ca-827d2e86367f] Cannot reboot instance: Unable to create tap device tape10b9639-d8: Device or resource busy
```

<!-- more -->

用 ifconfig 與 ip 指令都可以看到該 tap 存在

```
root@compute-14:~# ifconfig
tape10b9639-d8 Link encap:Ethernet  HWaddr fe:16:3e:cb:8f:29  
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:513131 errors:0 dropped:0 overruns:0 frame:0
          TX packets:691944 errors:0 dropped:9397 overruns:0 carrier:0
          collisions:0 txqueuelen:500
          RX bytes:94571524 (94.5 MB)  TX bytes:206871945 (206.8 MB)
```

```
root@compute-14:~# ip a | grep tape10b9639-d8
77: tape10b9639-d8: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast state DOWN qlen 500
```

不過 ovs-vsctl show 倒是沒有看到這個 tap device

----

直接使用 `ifconfig <tap name> down` 是沒有辦法讓這張網卡消失，在 StackOverflow 上看到一篇[參考文章](http://stackoverflow.com/questions/17529345/ubuntu-remove-network-tap-interface)

試了一下文章中提到 ip link set 的方式，結果沒效

```
root@compute-14:~# ip link set tape10b9639-d8 down
root@compute-14:~# ip a | grep tape10b9639-d8
77: tape10b9639-d8: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast state DOWN qlen 500
```

不過倒是讓我發現 `ip link delete <tap name>` 可以達成

# Solution


```
root@compute-14:~# ip link delete tape10b9639-d8
root@compute-14:~# ip a | grep tape10b9639-d8
```

之後再次使用 nova reboot 指令就沒有問題囉
