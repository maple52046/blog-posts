---
title: "Ceph pgs stuck in 'incomplete' state ,ops blocked"
date: 2015-02-11 17:46:53
categories: Storage
tags: Ceph
aliases:
  - 2015/02/11/Ceph-pgs-stuck-in-incomplete-state-and-ops-blocked/
---

Ceph OSD 又再次發生 [disk failure](http://worldend.logdown.com/posts/251761-ceph-osd-a-copy-of-the-executable-or-objdump-rds-executable-is-needed-to-interpret-this)，結果在手動修復硬碟時操作不當，整個 disk partition table 都消失了
即使把備份的 disk patition table 寫回去之後，依然無法解決問題。

無奈之下，硬是將 Ceph cluster 開啟 (少一個 OSD)

執行 `ceph health detail` 得到以下狀態:

```
pg 0.1f is stuck inactive since forever, current state incomplete, last acting [3]
pg 0.1f is stuck unclean since forever, current state incomplete, last acting [3]
pg 0.1f is incomplete, acting [3]
32 ops are blocked > 32.768 sec on osd.3
32 ops are blocked > 32.768 sec on osd.3
1 osds have slow requests
```

<!-- more -->

按照官網的[教學](http://docs.ceph.com/docs/master/rados/troubleshooting/troubleshooting-pg/#failures-osd-peering)，先執行 `ceph pg 01.f query`，看到 pg 的資訊如下(節錄):

```
{ "state": "incomplete",
  ...
  "recovery_state": [
       { "name": "Started\/Primary\/Peering",
         "enter_time": "2012-03-06 14:40:16.169659",
         "probing_osds": [
               0,
               1,
               2,
               3],
         "blocked": "",
         "down_osds_we_would_probe": [],
         "peering_blocked_by": []
}
```

與官網不同的是，這個 pg 並沒有顯示 unfound object，所以執行 `ceph pg 0.1f mark_unfound_lost revert` 只會出現 **pg has no unfound objects**

上網 google 了一下類似問題:

* [[ceph-users] PG down & incomplete](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2013-May/021095.html)
* [[ceph-users] HEALTH_WARN 4 pgs incomplete; 4 pgs stuck inactive; 4 pgs stuck unclean](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2014-August/042096.html)
* [pgs stuck in 'incomplete' state, blocked ops,	query command hangs](http://www.spinics.net/lists/ceph-users/msg12588.html)
* [[ceph-users] Constant slow / blocked requests with otherwise healthy cluster](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2013-November/035826.html)
* [[ceph-users] HEALTH_WARN 4 pgs incomplete; 4 pgs stuck inactive; 4 pgs stuck unclean](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2014-August/042225.html)

Google 到的解法大多都是:

1. `ceph pg 0.1f mark_unfound_lost revert`
2. `ceph pg force_create_pg 0.1f`
3. Shutdown Ceph OSD 3
4. `ceph osd lost 4 --yes-i-really-mean-it`

但是很遺憾，怎麼做就是沒有任何效果。

如果執行 `ceph pg scrub 0.1f` 會得到 **instructing pg 0.1f on osd.3 to scrub**，然後就沒下文了。而 `deep-scrub` 與 `repair` 也是一樣的狀況。

正當要準備放棄時，忽然靈機一動，調整了一下指令的順序

## Solution

根據 `ceph pg 01.f query` 得到的結果中得知，pg 0.1f 是存在於 0,1,2,3 這四個 OSD 上:

```
recovery_state": [
       { "name": "Started\/Primary\/Peering",
         "enter_time": "2012-03-06 14:40:16.169659",
         "probing_osds": [
               0,
               1,
               2,
               3],
        }
]
```

因此**第一步就是關閉這四個 OSD**

```
ssh node01 "stop ceph-osd id=0"
ssh node01 "stop ceph-osd id=1"
ssh node02 "stop ceph-osd id=2"
ssh node02 "stop ceph-osd id=3"
```

**第二步直接執行 `ceph osd lost`**，一樣也是四個 OSD 都要做:

```
ceph osd lost 0 --yes-i-really-mean-it
ceph osd lost 1 --yes-i-really-mean-it
ceph osd lost 2 --yes-i-really-mean-it
ceph osd lost 3 --yes-i-really-mean-it
```

這時候稍微等一段時間(大概5分鐘)，再次執行 `ceph health detail` 後，發現:
```
pg 0.1f is stuck inactive since forever, current state incomplete, last acting []
```
差別在於原本是 **last acting[3]**，代表最後一次是在 OSD 3 上面動作；而現在 pg 0.1f 並沒有在任何一個 OSD 上有動作。

> **PS: 此狀態是憑印象寫的，不是非常確定。但是`ceph osd lost`勢必一定要執行**

**第三步就是執行 `ceph pg force_create_pg 0.1f`**

然後，神奇的事情來了，過沒幾分鐘後再次執行 `ceph health detail`，就沒有再看到任何 pg 0.1f 的錯誤訊息

**第四步啟動原先四個 OSD**

```
ssh node01 "start ceph-osd id=0"
ssh node01 "start ceph-osd id=1"
ssh node02 "start ceph-osd id=2"
ssh node02 "start ceph-osd id=3"
```

接下來只需要讓 Ceph 同步一段時間，再次執行 `ceph health detail`，就可以得到 **HEALTH_OK**

收工!!!~

----

利用以上的步驟，雖然終於讓 Ceph 回復到正常狀態，但是與 pg 0.1f 有關的檔案幾乎都是損毀的狀態。因為我是 OpenStack  使用 Ceph，最直觀的影響就是很多 image、instance 無法正常開機。然而本次錯誤狀況其實是可以避免的，原因在於之前將 replication 設為 1。因此一個 OSD 掰掰了，上面的資料變得無法救援，也算是自作自受吧 :'(

----

## 後紀

經過這將近半年來各種狀況的考驗，目前對於 Ceph 有一些心得:

1. OSD 不要架設在 LVM 上。在網路看到很多文章都說是不要安裝在 raid 上，因為 Ceph 已經有自己的 fault torolence。但是根據目前的經驗來看，最好連 LVM 也不要，直接使用一整顆硬碟才是最好的方案。

2. Data replication 最好不要設成 1。否則，天有不測風雲...。即使要儲存的資料不是這麼重要，但是為了讓 Ceph 能正常運作，replication 建議保留原本的預設值(default is 2)。

3. Ceph cluster 最好要區分 public network 與 cluster network。雖然 cluster network 只有 OSDs 會用到，而且使用率很低。但是在 production 環境上，為了不影響其他 service (例如 OpenStack)，最好將其獨立出來。

4. Disk partition 最好也要做個備份，還有 Ceph 本身的檔案 (/var/lib/ceph/*)

另外，網路上有不少討論 xfs 與 ext4 到底哪個比較適合 Ceph。XFS 似乎在多線程同時讀寫上比 ext4 好，因此才會有 XFS 比 ext4 更適合 Ceph 一說。下次可以試試。
