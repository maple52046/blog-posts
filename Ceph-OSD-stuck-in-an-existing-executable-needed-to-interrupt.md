---
title: Ceph OSD stuck in an existing executable needed to interrupt.
date: 2015-01-19 11:39:36
categories: Storage
tags: Ceph
aliases:
  - 2015/01/19/Ceph-OSD-stuck-in-an-existing-executable-needed-to-interrupt/
---

上週 Ceph cluster 掛掉，結果是某個 OSD 一直起不來。
在 `/var/log/ceph/ceph-osd.X.log` 裡面看到了以下錯誤訊息

```
ceph version 0.80.5 (38b73c67d375a2552d8ed67843c8a65c2c0feba6)
1: (FileStore::lfn_open(coll_t, ghobject_t const&, bool, std::tr1::shared_ptr<FDCache::FD>*, std::tr1::shared_ptr<CollectionIndex::Path>*, std::tr1::shared_ptr<CollectionIndex>*)+0x4e6) [0x888926]
2: (FileStore::_touch(coll_t, ghobject_t const&)+0x18b) [0x88effb]
3: (FileStore::_do_transaction(ObjectStore::Transaction&, unsigned long, int, ThreadPool::TPHandle*)+0x48f6) [0x899856]
4: (FileStore::_do_transactions(std::list<ObjectStore::Transaction*, std::allocator<ObjectStore::Transaction*> >&, unsigned long, ThreadPool::TPHandle*)+0x74) [0x89b204]
5: (JournalingObjectStore::journal_replay(unsigned long)+0x886) [0x8af6e6]
6: (FileStore::mount()+0x30c2) [0x883052]
7: (OSD::do_convertfs(ObjectStore*)+0x1a) [0x61a11a]
8: (main()+0x1d88) [0x602f98]
9: (__libc_start_main()+0xed) [0x7f70e698276d]
10: /usr/bin/ceph-osd() [0x607229]
NOTE: a copy of the executable, or `objdump -rdS <executable>` is needed to interpret this.
```

<!-- more -->

因為 Ceph 掛掉時，剛好正在 VM 上安裝一些東西
原本以為是 VM 卡在要做 I/O 才當掉
但是把所有 server 都重開了以後，發現問題依然沒有解決。

Google 一番之後，看到有人說，升級可以解決這個問題
於是將 Ceph 從 Firefly upgrade 到 Giant，但是很遺憾的是依然沒有解決問題。

新增了一個 OSD 進去，天真的想說如果變成 1/4 的 OSD 掛掉，那是否可以讓 Ceph 繼續運作呢，結果還是失敗。

忽然想起了很久以前看到一篇文章，上面說 Ceph OSD 最好不要安裝在 raid 上
因為 Ceph 本身已經有 fault tolerance 了，如果硬碟發生錯誤，則兩者的 fault tolerance 將會互相影響。

雖然我的 OSD 不是建立在 raid 上，但是是在 LVM 上
於是，使用了 e2fsck 去檢查並修復，發現了兩個 block 壞了
修復完之後，再重新啟動 OSD
然後...然後 OSD 終於歸隊拉!!!

# 結論

LVM 與 Raid 本身就有自己的錯誤檢查機制。這些機制會與 Ceph 本身的的容錯機制互相衝突。
因為 Ceph OSD 最好還是一個 OSD 就一個硬碟。
