---
title: 'Ceph PG stuck in degraded + undersized'
date: 2015-12-17 17:08:53
categories: Storage
tags: Ceph
aliases:
  - 2015/12/17/Ceph-PG-stuck-stuck-in-degraded-and-undersized/
---

執行 `ceph -s` 出現：

```
cluster 1a1d374a-c6e9-48cb-9b45-525a6fdaa91e
 health HEALTH_WARN
        64 pgs degraded
        64 pgs stale
        64 pgs stuck degraded
        64 pgs stuck stale
        64 pgs stuck unclean
        64 pgs stuck undersized
        64 pgs undersized
 monmap e1: 1 mons at {twin-storage-01=172.16.91.1:6789/0}
        election epoch 2, quorum 0 twin-storage-01
 mdsmap e5: 1/1/1 up {0=twin-storage-01=up:active}
 osdmap e92: 7 osds: 7 up, 7 in
  pgmap v685: 832 pgs, 7 pools, 43573 kB data, 38 objects
        7491 MB used, 14889 GB / 14896 GB avail
             769 active+clean
              37 stale+active+undersized+degraded+remapped
              26 stale+active+undersized+degraded
```

很多 pg 卡在 `degraded` + `undersized` 狀態。
<!-- more -->
執行 `ceph health detail` 看到詳細一點的資訊：

```
pg 0.2c is stuck stale for 59834.916633, current state stale+active+undersized+degraded, last acting [0]
pg 0.2b is stuck stale for 59834.916635, current state stale+active+undersized+degraded, last acting [0]
pg 0.2a is stuck stale for 59834.916637, current state stale+active+undersized+degraded+remapped, last acting [0]
pg 0.29 is stuck stale for 59834.916638, current state stale+active+undersized+degraded, last acting [0]
pg 0.28 is stuck stale for 59834.916640, current state stale+active+undersized+degraded, last acting [0]
pg 0.27 is stuck stale for 59834.916642, current state stale+active+undersized+degraded+remapped, last acting [0]
pg 0.26 is stuck stale for 59834.916644, current state stale+active+undersized+degraded+remapped, last acting [0]
pg 0.25 is stuck stale for 59834.916645, current state stale+active+undersized+degraded+remapped, last acting [0]
pg 0.24 is stuck stale for 59834.916647, current state stale+active+undersized+degraded+remapped, last acting [0]
```

想要利用 `ceph pg <pgid> query` 查看 pg 的詳細資訊，卻出現 error：

```
$ ceph pg 0.24 query
Error ENOENT: i don't have pgid 0.24
```

猜測問題產生的原因，可能是我在建置這一套 Ceph cluster 時，曾經把所有的 OSD 都移出重建。可能移除時沒有把資料清乾淨；或是我移除的方法不對，...等原因。

----

# Solution

解決方法就是用 `ceph pg force_creat_pg <pgid>` 去覆蓋那個有問題的 pg

```
$ ceph pg force_create_pg 0.24
pg 0.24 now creating, ok
```

這個 pg 就會轉成 `creating`，過一段時間，等 `creating`完成之後，就可以 query 出那個 pg 的資訊：

```
$ ceph pg 0.24 query
{
    "state": "active+clean",
    "snap_trimq": "[]",
    "epoch": 94,
    "up": [
        2,
        4
    ],

    ... (省略)...

    "agent_state": {}
}
```

同時，`ceph -s`也可以看到少了有狀況的 pg：
```
cluster 1a1d374a-c6e9-48cb-9b45-525a6fdaa91e
 health HEALTH_WARN
        63 pgs degraded
        63 pgs stale
        63 pgs stuck degraded
        63 pgs stuck stale
        63 pgs stuck unclean
        63 pgs stuck undersized
        63 pgs undersized
 monmap e1: 1 mons at {twin-storage-01=172.16.91.1:6789/0}
        election epoch 2, quorum 0 twin-storage-01
 mdsmap e5: 1/1/1 up {0=twin-storage-01=up:active}
 osdmap e92: 7 osds: 7 up, 7 in
  pgmap v685: 832 pgs, 7 pools, 43573 kB data, 38 objects
        7491 MB used, 14889 GB / 14896 GB avail
             769 active+clean
              37 stale+active+undersized+degraded+remapped
              26 stale+active+undersized+degraded
```


----

如果有問題的 pg 數量很多，可以用 for loop 去跑：
```
for pg in `ceph health detail | grep "stale+active+undersized+degraded" | awk '{print $2}' | sort | uniq`;
do
  ceph pg force_create_pg $pg
done
```

用 for loop 跑指令下太快，可能會變成以下狀況：

```
cluster 1a1d374a-c6e9-48cb-9b45-525a6fdaa91e
 health HEALTH_WARN
        63 pgs stuck inactive
        63 pgs stuck unclean
 monmap e1: 1 mons at {twin-storage-01=172.16.91.1:6789/0}
        election epoch 2, quorum 0 twin-storage-01
 mdsmap e5: 1/1/1 up {0=twin-storage-01=up:active}
 osdmap e92: 7 osds: 7 up, 7 in
  pgmap v892: 832 pgs, 7 pools, 45412 kB data, 42 objects
        7496 MB used, 14889 GB / 14896 GB avail
             769 active+clean
              63 creating
```

`ceph health detal` 出現：

```
pg 0.31 is stuck inactive since forever, current state creating, last acting []
```

這時先不要急，放著讓他處理一段時間即可。
