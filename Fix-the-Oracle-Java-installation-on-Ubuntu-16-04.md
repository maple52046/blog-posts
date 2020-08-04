---
title: Fix the Oracle Java installation on Ubuntu 16.04
date: 2017-10-19 08:38:53
categories: Linux
tags:
  - Java
  - Ubuntu
  - apt
  - deb
aliases:
  - 2017/10/19/Fix-the-Oracle-Java-installation-on-Ubuntu-16-04/
---

在 Ubuntu 透過 [WebUpd8 team ppa](https://launchpad.net/~webupd8team/+archive/ubuntu/java) 安裝 Java：

```
sudo add-apt-repository ppa:webupd8team/java
sudo apt update
sudo apt install oracle-java9-installer
```

安裝的過程中發生 404 Not found：

```
Downloading Oracle Java 9...
--2017-10-19 08:41:59--  http://download.oracle.com/otn-pub/java/jdk/9+181/jdk-9_linux-x64_bin.tar.gz
Resolving download.oracle.com (download.oracle.com)... 210.61.248.163, 210.61.248.216
Connecting to download.oracle.com (download.oracle.com)|210.61.248.163|:80... connected.
HTTP request sent, awaiting response... 302 Moved Temporarily
Location: https://edelivery.oracle.com/otn-pub/java/jdk/9+181/jdk-9_linux-x64_bin.tar.gz [following]
--2017-10-19 08:41:59--  https://edelivery.oracle.com/otn-pub/java/jdk/9+181/jdk-9_linux-x64_bin.tar.gz
Resolving edelivery.oracle.com (edelivery.oracle.com)... 104.116.18.92, 2600:1417:1b:188::2d3e, 2600:1417:1b:184::2d3e
Connecting to edelivery.oracle.com (edelivery.oracle.com)|104.116.18.92|:443... connected.
HTTP request sent, awaiting response... 302 Moved Temporarily
Location: http://download.oracle.com/otn-pub/java/jdk/9+181/jdk-9_linux-x64_bin.tar.gz?AuthParam=1508373839_4021b0b2a88845635225f537490b9b5a [following]
--2017-10-19 08:42:00--  http://download.oracle.com/otn-pub/java/jdk/9+181/jdk-9_linux-x64_bin.tar.gz?AuthParam=1508373839_4021b0b2a88845635225f537490b9b5a
Connecting to download.oracle.com (download.oracle.com)|210.61.248.163|:80... connected.
HTTP request sent, awaiting response... 404 Not Found
2017-10-19 08:42:01 ERROR 404: Not Found.

download failed
Oracle JDK 9 is NOT installed.
dpkg: error processing package oracle-java9-installer (--configure):
 subprocess installed post-installation script returned error exit status 1
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

在等待官方之餘，嘗試先自己修改 deb 來解決這個問題。

<!--more-->

---

# Solution

## Install tools

先安裝 tool：

```
sudo apt install apt-src
```

接下來，修改 `/etc/apt/sources.list.d/webupd8team-ubuntu-java-xenial.list`，取消 deb-src 的註解：

```text
deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main
deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main
```

然後更新 apt-src 的資料庫：

```
sudo apt-src update
```

## Download deb source

下載 oracle-java9-installer 的 deb source：

```
mkdir ~/oracle-java
cd ~/oracle-java
sudo apt-src install oracle-java9-installer
```

這邊還是會再出現一次錯誤，不過可以忽略。

![](https://i.imgur.com/k8hI5n8.png)

## Modify deb file

- Step 1:

先到 [Oracle 官網](http://www.oracle.com/technetwork/java/javase/downloads/jdk9-downloads-3848520.html) 去確認安裝檔的路徑：

![](https://i.imgur.com/YN0S4mp.png)

(安裝檔 `jdk-9.0.1_linux-x64_bin.tar.gz` 的下載位址為：http://download.oracle.com/otn-pub/java/jdk/9.0.1+11/jdk-9.0.1_linux-x64_bin.tar.gz)

編輯 ~/oracle-java/oracle-java9-installer-9b181/debian/oracle-java9-installer.postinst:

  - 將 `JAVA_VERSION_MAJOR` (line 68) 修改成 **`9.0.1`**
  - 將 `J_DIR` (line 71) 修改成 **`jdk-9.0.1`**
  - 將 `JAVA_VERSION_MINOR` (line 69) 修改成 **`11`**

Ex:

```bash
if [ ! $arch = "arm" ]; then
  JAVA_VERSION_MAJOR=9 #
  JAVA_VERSION_MINOR=181 #must be modified for each release
  JAVA_VERSION_DATE="04_nov_2015" #no longer needed with b95
  J_DIR=jdk-9 #must be modified for each release
  FILENAME=jdk-${JAVA_VERSION_MAJOR}_linux-${dld}_bin.tar.gz # jdk-9_linux-x64_bin.tar.gz
  PARTNER_URL=http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}+${JAVA_VERSION_MINOR}/$FILENAME
else
  JAVA_VERSION_MAJOR=9.0.1 #
  JAVA_VERSION_MINOR=11 #must be modified for each release
  JAVA_VERSION_DATE="04_nov_2015" #no longer needed with b95
  J_DIR=jdk-9.0.1 #must be modified for each release
  FILENAME=jdk-${JAVA_VERSION_MAJOR}_linux-${dld}_bin.tar.gz # jdk-9+109_linux-arm64-vfp-hflt_bin.tar.gz
  PARTNER_URL=http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}+${JAVA_VERSION_MINOR}/$FILENAME
fi
```

- Step 2:

根據 [Oracle 官網]() 的 checksum 表：

![](https://i.imgur.com/K2ReH3E.png)

修改 **SHA256SUM_TGZ** (line 24) 的值：

Ex:

```bash
case $(dpkg --print-architecture) in
'i386'|'i586'|'i686')
  #arch=i386; dld=x86;
  #SHA256SUM_TGZ="ba0c77644ece024cdb933571d79f0f035e91a9c9ab70de9c82446c9fbd000c97" #must be modified for each release
  echo "Error. Oracle Java 9 does not support 32bit."
  ;;
'amd64'  ) arch=amd64; dld=x64;
  SHA256SUM_TGZ="2cdaf0ff92d0829b510edd883a4ac8322c02f2fc1beae95d048b6716076bc014" #must be modified for each release
  ;;
```

## Build deb

接下來開始 build deb 檔：

```bash
apt-src build oracle-java9-installer
```

完成之後，會在目錄下看到三個 deb 檔：

- oracle-java9-installer_9b181-1~webupd8~2_amd64.deb
- oracle-java9-unlimited-jce-policy_9b181-1~webupd8~2_amd64.deb
- oracle-java9-set-default_9b181-1~webupd8~2_amd64.deb

## Install Oracle Java

```bash
sudo dpkg -i oracle-java9-installer_9b181-1~webupd8~2_amd64.deb
sudo apt install oracle-java9-set-default
```
