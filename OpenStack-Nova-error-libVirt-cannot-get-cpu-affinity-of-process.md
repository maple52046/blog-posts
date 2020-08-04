---
title: 'OpenStack Nova error: libVirt cannot get CPU affinity of process 30619'
date: 2015-02-11 15:55:00
categories: OpenStack
tag:
  - OpenStack
  - libVirt
aliases:
  - 2015/02/11/OpenStack-Nova-error-libVirt-cannot-get-cpu-affinity-of-process/
---

Nova-compute 開不起來，在 log 中發現:

```
2015-02-11 15:34:05.511 30827 TRACE nova   File "/usr/lib/python2.7/dist-packages/eventlet/tpool.py", line 187, in doit
2015-02-11 15:34:05.511 30827 TRACE nova     result = proxy_call(self._autowrap, f, *args, **kwargs)
2015-02-11 15:34:05.511 30827 TRACE nova   File "/usr/lib/python2.7/dist-packages/eventlet/tpool.py", line 147, in proxy_call
2015-02-11 15:34:05.511 30827 TRACE nova     rv = execute(f,*args,**kwargs)
2015-02-11 15:34:05.511 30827 TRACE nova   File "/usr/lib/python2.7/dist-packages/eventlet/tpool.py", line 76, in tworker
2015-02-11 15:34:05.511 30827 TRACE nova     rv = meth(*args,**kwargs)
2015-02-11 15:34:05.511 30827 TRACE nova   File "/usr/lib/python2.7/dist-packages/libvirt.py", line 2096, in vcpus
2015-02-11 15:34:05.511 30827 TRACE nova     if ret == -1: raise libvirtError ('virDomainGetVcpus() failed', dom=self)
2015-02-11 15:34:05.511 30827 TRACE nova libvirtError: cannot get CPU affinity of process 30619: No such process
2015-02-11 15:34:05.511 30827 TRACE nova
```

<!-- more -->

雖然看不太懂為什麼會出現這種錯誤，不過猜測大概與早上將 compute node 強制重開機有關。

看一下 `virsh list` 結果一大堆 VM 卡在 *running* 或 *shutdown* 的狀態。
但是並沒有任何 KVM process 在 running。

雖然不知道為什麼，但是我的直覺告訴我，只要解決 libVirt 這個狀態，就能解決 nova-compute 的問題

# Solution

找了一下，發現在 **/var/run/libvirt/qemu/** 這個資料夾中，有很多 instance-XXXXX.pid 與 instance-XXXXX.xml

```
root@compute-02: /var/run/libvirt/qemu# ls
instance-00000093.pid  instance-000002bb.xml  instance-00000307.pid  instance-00000327.xml  instance-00000340.pid
instance-00000093.xml  instance-000002be.pid  instance-00000307.xml  instance-0000032b.pid  instance-00000340.xml
instance-000000ac.pid  instance-000002be.xml  instance-00000322.pid  instance-0000032b.xml  instance-00000348.pid
instance-000000ac.xml  instance-000002c5.pid  instance-00000322.xml  instance-00000333.pid  instance-00000348.xml
instance-00000271.pid  instance-000002c5.xml  instance-00000325.pid  instance-00000333.xml  instance-00000357.pid
instance-00000271.xml  instance-000002c8.pid  instance-00000325.xml  instance-0000033d.pid  instance-00000357.xml
instance-000002bb.pid  instance-000002c8.xml  instance-00000327.pid  instance-0000033d.xml
```

這些檔案應該是要 instance running 時，libVirt 自己產生。我猜測 `virsh list` 應該是會來這個目錄讀取檔案。反正現在也沒有 VM 正在 running，乾脆把他們全部砍掉。
```
root@compute-02: /var/run/libvirt/qemu# rm -rf *
```

接下來重啟 libVirt ，然後就發現 `virsh list` 恢復到原本的狀態。
```
root@compute-02:~# service libvirt-bin restart
libvirt-bin stop/waiting
libvirt-bin start/running, process 29451
root@compute-02:~# virsh list
 Id    Name                           State
----------------------------------------------------

root@compute-02:~#
```

接著啟動 nova-compute 就可以恢復運作囉。
```
root@compute-02:~# service nova-compute start
nova-compute start/running, process 32664
root@compute-02:~# service nova-compute status
nova-compute start/running, process 32664
```
