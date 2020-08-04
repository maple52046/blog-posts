---
title: Compile OpenWRT with Open vSwitch
date: 2015-03-05 17:05:00
categories: Network
tags:
 - OpenWRT
 - Open vSwitch
toc: true
aliases:
  - 2015/03/05/Compile-OpenWRT-with-Open-vSwitch/
---

雖然說在 google 上很多資料都說，Open vSwitch 已經加入到 OpenWRT 的 packages repository，而且在 GitHub 上也有看到 [Open vSwitch](https://github.com/openwrt/packages/tree/master/net/openvswitch) 的身影。但是實際上，在官方的 [repository](https://downloads.openwrt.org/barrier_breaker/14.07/) 中並沒有 Open vSwitch。因此決定自行編譯。

# **實驗環境**

- 編譯環境
    - Operating System: Ubuntu 14.04 x64
    - OpenWRT version: 14.07
    - Open vSwitch version: 2.3.1

- 路由器
    - [TP-Link TL-WR1043ND](http://www.tp-link.tw/products/details/?model=TL-WR1043ND)

<!-- more -->
----

# **開始編譯**

- **首先要先安裝編譯環境**
```
sudo apt-get -y install build-essential subversion git-core libncurses5-dev zlib1g-dev gawk flex quilt libssl-dev xsltproc libxml-parser-perl unzip
```

- **下載 source code (stable version, 14.07)**
```
user@localhost:~$ git clone git://git.openwrt.org/14.07/openwrt.git
```

- **建立 feeds.conf**
```
user@localhost:~$ cd openwrt
user@localhost:~/openwrt$ mv feeds.conf.default feeds.conf
```

- **執行 `feeds update`**
```
user@localhost:~/openwrt$ ./scripts/feeds update -a
```
    指令結束後，會看到當前目錄下多了一個資料夾，名稱為 **feeds**，裡面將會根據 feeds.conf 的每個項目，一個項目將會產生兩個目錄、一個 soft-link，例如:
```
user@localhost:~/openwrt$ ls -lh feeds/
drwxrwxr-x 13 user user 4.0K  3月  9 14:24 packages
lrwxrwxrwx  1 user user   25  3月  9 14:52 packages.index -> packages.tmp/.packageinfo
drwxrwxr-x  3 user user 4.0K  3月  9 14:24 packages.tmp
```

- **將 Open vSwitch repo 新增到 feeds.conf 中**

    先檢查看看 packages 與 oldpackages 裡面是否有 openvswitch:
```
user@localhost:~/openwrt$ ls feeds/packages/net/ | grep openvswtich
user@localhost:~/openwrt$ ls feeds/oldpackages/net/ | grep openvswitch
```

    如果有，則跳過此步驟；反之，則新增 Open vSwitch repo:
```
user@localhost:~/openwrt$ echo 'src-git openvswitch git://github.com/pichuang/openvwrt.git' >> feeds.conf
user@localhost:~/openwrt$ ./script/feeds update -a
```

    然後打上 libatomic patch:
```
user@localhost:~/openwrt$ wget https://gist.githubusercontent.com/pichuang/7372af6d5d3bd1db5a88/raw/4e2290e3e184288de7623c02f63fb57c536e035a/openwrt-add-libatomic.patch -q -O - | patch -p1
```

    執行完之後，在 feeds 資料夾裡面看到 openvswitch:
```
user@localhost:~/openwrt$ ls -lh feeds/ | grep openvswitch
drwxrwxr-x  4 user user 4.0K  3月  9 14:52 openvswitch
lrwxrwxrwx  1 user user   28  3月  9 14:52 openvswitch.index -> openvswitch.tmp/.packageinfo
drwxrwxr-x  3 user user 4.0K  3月  9 14:52 openvswitch.tmp
```

    - **更改 Open vSwitch 版本**
        原本的 maintainer 已經不再維護了，因此版本還是 2.3.0。如果想要更改版本，則編輯 `feeds/openvswitch/openvswitch/Makefile`
```make feeds/openvswitch/openvswitch/Makefile
PKG_RELEASE:=1
PKG_VERSION:=2.3.0
```
        修改 `PKG_RELEASE` 與 `PKG_VERSION` 這兩個值:
```make feeds/openvswitch/openvswitch/Makefile
PKG_RELEASE:=0
PKG_VERSION:=2.3.1
```
        然後再次執行 `feeds update`
```
user@localhost:~/openwrt$ ./scripts/feeds update openvswitch
```
        這時候就可以看到 `feeds/openvswitch.index` 裡面，Open vSwitch 的版本就改成了 **2.3.1-0**
```
Source-Makefile: feeds/openvswitch/openvswitch/Makefile
Package: openvswitch-ipsec
Version: 2.3.1-0
```

- **執行 `feeds install`**
```
user@localhost:~/openwrt$ ./scripts/feeds install -a
```

- **執行 `make menuconfig`**
```
user@localhost:~/openwrt$ make menuconfig
```
    接下來會開啟編譯選單:<br />
![make_menuconfig.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/sjalQsXQQmSupf39sAid_make_menuconfig.jpg)

    利用上下左右、M、空白鍵來切換與選擇不同的編譯選項。在選擇編譯選項時，原則上是:
    1. 系統預設值保留不變，除非要拿掉甚麼功能，ex: ppp。
    2. 盡量將需要編譯的項目選擇為 **M**，而不是 <strong>*</strong>，除非認定必要存在。

    選擇的過程中或是離開前，選擇 **save** 可以將選項儲存起來，檔名保留為 **.config** (檔名不可以變更)。<br />
![save_config.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/S0tZR8TwRa2v87AuN92F_save_config.jpg)

    - **Target Profile**

        如果 OpenWRT 是要自己使用的，可以指定要編譯哪些 firmware，這樣可以節省不少編譯時間。<br />
        選擇 `Target Profile` -> 選擇指定的 device 名稱，例如: TP-Link TL-W1043N/ND<br />
![custom_profile.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/T7gxdCp0SAmLXpoyZvnH_custom_profile.jpg)

    - **Open vSwitch**
        1. 選擇 `Network` -> `openvswitch-common`、`openvswitch-ipsec`、`openvswitch-switch` <br /><br />
![choice_openvswitch.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/KlkgzKJGSVSWdkhAGBOL_choice_openvswitch.jpg)<br />
        2. 選擇 `Advanced configuration options (for developers)` -> `Target Options` -> **取消勾選** `Build packages with MPIS16 instructions`<br />
        3. 選擇 `Advanced configuration options (for developers)` -> `Toolchain Options` -> `Binutils Version` ->`Linaro binutils 2.24`

    - **Luci** (Option.)

        如果想要網頁管理介面，可以選擇加裝 luci<br />
        選擇 `Luci` -> `Collections` -> `luci`<br />
![luci.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/O6dnoyPcQ0KoIldgbv3Y_luci.jpg)

- **Compile OpenWRT**
    選擇完編譯選項，並將設定值儲存為 **.config** 之後，接下來執行:<br />
```
user@localhost:~/openwrt$ echo '#CONFIG_KERNEL_BRIDGE is not set' >> .config
```
    這個步驟再每一次 `make menuconfig` 之後都要執行一次<br />
    然後開始編譯 OpenWRT<br/>
```
user@localhost:~/openwrt$ make V=s
```
    編譯完成後的檔案會放在 `openwrt/bin/`

<a name="CustomRepository"></a>
- **架設 repository**
為了方便燒 OpenWRT 到 AP 裡面，可以順便架設 web server ，透過 HTTP 來提供 firmware 載點。以 nginx 為範例:

    - 安裝 web server
```
user@localhost:~/openwrt$ sudo apt-get -y install nginx
```

    - 編輯 `/etc/nginx/sites-available/default`，在 **server** 的設定裡面，新增以下 location 片段:
```nginx
location /openwrt/downloads {
    alias /home/user/openwrt/bin;
    autoindex on;
}
```
        請根據喜好與系統狀態去設定:
        - URL: `/openwrt/downloads`
        - PATH (編譯後的 firmware 放置位置): `/home/user/openwrt/bin`

    - 重啟 nginx
```
user@localhost:~/openwrt$ sudo service nginx restart
```

    然後打開瀏覽器，輸入網址: `http://<<your ip>>/openwrt/downloads`，應該要出現類似以下的畫面:<br />
![custom_repository_1.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/2NycfcuTRCiuY67uumXA_custom_repository_1.jpg)<br />
    點擊 **ar71xx** 後:<br />
![custom_repository_2.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/Az6shvIdRqyXaK4wDUkV_custom_repository_2.jpg)

----

# **使用 OpenWRT**
接下來就可以利用 Luci 、[Sysupgrade 指令](http://wiki.openwrt.org/doc/howto/generic.sysupgrade)、AP 原廠的管理介面等方法去燒 OpenWRT。<br />
OpenWRT 第一次開機時，要使用 telnet 的方式連入
```
telnet 192.168.1.1
```
進入系統之後，更換 root 密碼。登出之後，接下來就改用 ssh (或是 Luci) 登入。

----

# Reference

- [OpenFlow 1.3 for OpenWRT on TL-1043ND with OVS](http://linton.tw/2014/05/13/openflow-13-for-openwrt-on-tl-1043nd-with-open-vswitch/)
- [編譯 OpenWrt](http://roan.logdown.com/posts/165911-compiled-openwrt)
- [Porting OpenvSwitch to OpenWrt](http://roan.logdown.com/posts/208499-openvswitch-lab-5-porting-openvswitch-to-openwrt)


<!----  以下為保留段落,此區段文章不顯示
#### OpenSSH server

選擇 `Network` -> `SSH` -> 選擇 `openssh-server` 與 `openssh-server-pam`

![openssh-server.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/UbXhlSl0Sxy3p5GzSK9f_openssh-server.jpg)

#### Editor (VIM)

選擇 `Utilities` -> `Editors` -> `vim-full`

![vim.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/7IVxT0wYSQqp1DO2KBrl_vim.jpg)

如果不太會需要編輯器的話，可以選擇 `vim` 或是 `vim-runtime`

#### Bash

選擇 `Utilities` -> `bash-completion`

![bash.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/2CF91bTR0CfxopFT9tIZ_bash.jpg)

#### 網路預設值 (Option.)

選擇 `Image configuration` -> `Preinit configuration options` 然後就可以看到最下面有 IP 預設值，如果有需要的可以修改:

![Preinit configuration options.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/aLk1ao2QcWcxmaT75dQo_Preinit%20configuration%20options.jpg)


#### WGet

選擇 `Network` -> `File Transfer` -> `wget`

![wget.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/Ds0o1GRCQiqBBkVTBvHv_wget.jpg)

#### Custom repositry (Option.)

預設的 repository 是 **http://downloads.openwrt.org/snapshots/trunk/%T/packages**
如果想要讓編譯好的 image 預設就使用自己的 repo 修改的方式如下:

選擇 `Image configuration` -> `Version configuration options` -> `Release repository`

![custom_repository_4.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/P4s3sy1Sg2H238TQwuoX_custom_repository_4.jpg)

不知道為甚麼，這個參數我無法直接編輯，按 delete 鍵都沒有反應

目前解決的方法是:

1. 先儲存當前的設定值為 `.config`，然後離開 menucnofig
2. 編輯 `.config` 檔，找到 `CONFIG_VERSION_REPO="http://downloads.openwrt.org/snapshots/trunk/%T/packages"` 並修改成自己的 repository。例如: `CONFIG_VERSION_REPO="http://maple52046.twbbs.org/openwrt/downloads/%T/packages/"`

存檔，然後再次執行 `make menuconfig`，就可以看到預設的 repository 已經改成自己的網址

![custom_repository_3.jpg](http://user-image.logdown.io/user/10779/blog/10403/post/256561/wTfI8T1Qw2oilVJbOjAi_custom_repository_3.jpg)

當然，別忘了編譯完成之後，還要[架設自己的 repository](#CustomRepository)。

----->
