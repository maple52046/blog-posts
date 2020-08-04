---
title: Setup Squid Transparent Proxy with Docker
date: 2015-11-19 17:09:00
categories: Container
tags:
  - Docker
  - Squid
aliases:
  - 2015/11/19/Setup-Squid-Transparent-Proxy-with-Docker/
---

Squid 作為 Transparent proxy 時，不但可以加快區域網絡內的速度、降低網路流量
還可以控管區網內是否要開放/封鎖網站~~(監看區網內誰在玩FB或是看色情網站XD)~~

通常 Transparent proxy 都會放在區網對外的那台 server 上，例如下圖：

<img class="center" src="http://user-image.logdown.io/user/10779/blog/10403/post/314634/N4mq1E37S767W0ZY5S3U_%E6%9C%AA%E5%91%BD%E5%90%8D.001.jpeg" alt="Squid-transparent-proxy.jpeg">

對於 Proxy , Transparent proxy 沒有概念的人，可以先看[鳥哥的文章][5]，這邊就很不負責的說不再贅述了((懶~

實驗環境
=======

| Item | Value |
|:----:|:-----:|
| OS | Gentoo |
| Docker version | 1.7.9 |
| Docker image | [sameersbn/squid:3.3.8-4][1] |

只要能跑 Docker, 實體機的 OS 是什麼都不重要 :D

<!-- more -->
----

安裝步驟
=======

## 1. 安裝 Docker

在 Gentoo 上安裝 Docker 會比較麻煩一些，要注意 kernel 有些功能要打開，其他就按照 [Docker 官方教學(For Gentoo)][2]即可。
至於其他 OS，就只好很不負責任的說，請參考 [Docker 官方教學](http://docs.docker.com/engine/installation/)安裝吧

## 2. 設定 Memory Disk (Option.)

既然 Squid 是一個 Proxy server，那通常就會有一個存放 cache 的地方。一般來說，如果 server 的 memory 夠大，將 cache 放到 memory 中是一個好選擇，可以讓效能快很多。

```
$ sudo mkdir /var/spool/squid3
$ echo "tmpfs /var/spool/squid3/ tmpfs defaults,size=256m,mode=1777 0 0" | sudo tee -a /etc/fstab
$ sudo mount /var/spool/squid3
```

ps: **`tee` 這個指令千萬別忘了加上 `-a`，免得 `/etc/fstab` 的內容會被覆蓋。**

> 手動掛載 `sudo mount -t tmpfs -o size=256M,mode=1777 tmpfs /var/spool/squid3`

## 3. 運行 Container

裝好 Docker 之後，接下來我們要運行 container。在網路上已經有安裝好的 docker image，可以直接從 docker.io 上下載。在這裡我選的是 [sameersbn/squid:3.3.8-4][1] 。

執行 `docker run` 指令，下載 docker image 並運行 Squid container：
```
$ sudo docker run -d --name squid3 --net host -v /var/log/squid3:/var/log/squid3 -v /var/spool/squid3:/var/spool/squid3 sameersbn/squid:3.3.8-4
```

- `-d` 是指 daemon mode, 也就是背景運作。
- `--name` 就是這個 container 的名稱。這個參數可以自由選擇要不要加，不加的話，docker 會直接以那一長串的 UUID 為名稱；**為了操作方便，建議指定 container 的名稱**
- `-net` 是 container 的網路模式。一般如果不加這個參數，預設是 bridge。如果不是 `host` mode 時，必須要加上 `-p 3128:3128` 或是自行設定 iptables nat rule 才能讓其他電腦連到 container。
- `-v` 是將實體機上的某個路徑，掛載到 container 的某個路徑。
  - `/var/log/squid3` 是 Squid 預設的 log 檔目錄。如果想要方便一點查看 log，建議可以掛載這個目錄。
  - `/var/spool/squid3` 是 Squid 預設的 cache 目錄。如果想要用 memory disk 來提高效能，記得做完第二步驟，並設定這個參數。
- `sameersbn/squid:3.3.8-4` 就是 docker image 名稱。

利用 `docker ps` 來檢查 container 是否有在運作：

```
sudo docker ps
CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS               NAMES
2d8d9f19b25d        sameersbn/squid:3.3.8-4   "/sbin/entrypoint.sh   2 hours ago         Up About an hour                        squid3
```

如果想要看 log，有兩種方式：

1. 如果 `docker run` 的參數有加上 `-v <<some path in host server>>:/var/log/squid3` 就可以在實體機上看到 log 檔。 在我的範例中是 `/var/log/squid3/cache.log`
2. 如果沒有掛載 log 路徑，你也可以執行 `docker logs <<container name>>` 查看 log。

## 4. 設定 Squid

Squid 的設定檔是 `squid.conf`，存放在 container 裡面。想要修改可以直接進入 container 裡面，或是執行 `sudo docker exec -it <<contianer name>> vi /etc/squid3/squid.conf`

如果你的 [Docker 是 1.8 以上的版本][3]，可以用 `docker cp` 將檔案下載到實體機上，編輯完後再傳回去。但可惜我不是。

為了圖方便，我將 `/etc/squid3` 整個資料夾下載到實體機，然後刪除舊的 container，重新建立新的 container 並加上掛載設定檔目錄的參數

```
$ sudo docker cp squid3:/etc/squid3 /etc/
$ sudo docker rm -f squid3
$ sudo docker run -d --name squid3 --net host -v /etc/squid3:/etc/squid3 -v /var/log/squid3:/var/log/squid3 -v /var/spool/squid3:/var/spool/squid3 sameersbn/squid:3.3.8-4
```

### 編輯 squid.conf

接下來就要編輯 squid.conf，以下先提供我的設定檔：

```
acl localnet src 10.0.0.0/8	    # RFC1918 possible internal network
acl localnet src 172.16.0.0/12	# RFC1918 possible internal network
acl localnet src 192.168.0.0/16	# RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines
acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http
acl CONNECT method CONNECT
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager
http_access deny to_localhost
http_access allow localnet
http_access allow localhost
http_access deny all
http_port 3128 transparent
https_port 3129 transparent cert=/etc/squid3/squid-proxy.crt key=/etc/squid3/squid-proxy.key ssl-bump
ssl_bump none all
```

- `acl localnet src 192.168.0.0/16`
  - `localnet` 是一個自訂的名稱
  - `src` 是指 request 的來源
  - `192.168.0.0/16` 也就是我們後端的 LAN。
- `http_port 3128 transparent`
  - `http_port` 就是設定 squid 負責 listen HTTP request 的 port。預設是 `3128`。
  - `transparent` 就是運作模式，因為我們要建置 transparent proxy，所以必須要加上這個參數。
- `https_port 3129 transparent cert=/etc/squid3/ssl/squid-proxy.crt key=/etc/squid3/ssl/squid-proxy.key ssl-bump`
  - `https_port` 顧名思義就是接受 https request 的 port。不能設定跟 `http_port` 一樣
  - `cert` 跟 `key` 必須要設定一對 self signed certificate 的路徑。
  - `ssl-bump` 設定 HTTPs 的加解密訊息要怎麼處理。這邊建議搭配 `ssl_bump none all` 這個設定，讓 client 直接讀取 Web server 的 certificate。

其他的參數可以不用動

### 產生 self signed certificate

首先必需先安裝 OpenSSL。在 Gentoo 上應該是 `dev-libs/openssl`;而 Ubuntu 上則只需要 `apt-get install openssl` 即可。

>產生出來的 certificate 必須要 Squid 能讀取到。因此，如果你有安裝我之前的步驟，將實體機上的某個資料夾掛載到 container，例如： `/etc/squid3` 之類，你就可以利用這些資料夾來傳檔案；
>若否，則建議直接在 container 中執行這個步驟。（例如可以 `docker exec -it squid3 openssl ......` 但是這樣路徑必須要是絕對路徑，例如 `/etc/squid3/squid-server.key`)

安裝好之後，接下來安裝以下指令步驟：

```
$ openssl genrsa -des3 -out squid-server.key 1024
Generating RSA private key, 1024 bit long modulus
............................++++++
...........................................................++++++
e is 65537 (0x10001)
Enter pass phrase for squid-server.key:
Verifying - Enter pass phrase for squid-server.key:
```

`pass phrase` 必須要打一串密碼進去，不可以留空

```
$ openssl req -new -key squid-server.key -out squid-server.csr
Enter pass phrase for squid-server.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:TW
State or Province Name (full name) [Some-State]:Taiwan
Locality Name (eg, city) []:Hsichu
Organization Name (eg, company) [Internet Widgits Pty Ltd]:NTHU
Organizational Unit Name (eg, section) []:SSLab
Common Name (e.g. server FQDN or YOUR name) []:maple52046.twbbs.org.tw
Email Address []:maple52046@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

後面 `extra` 這邊全部直接按 enter 跳過。

然後接下來還有兩個指令：
```
$ openssl rsa -in squid-server.key -out squid-proxy.key
Enter pass phrase for squid-server.key:
writing RSA key

$ openssl x509 -req -days 365 -in squid-server.csr -signkey squid-proxy.key -out squid-proxy.crt
Signature ok
subject=/C=TW/ST=Taiwan/L=Hsichu/O=NTHU/OU=SSLab/CN=maple52046.twbbs.org.tw/emailAddress=maple52046@gmail.com
Getting Private key
```

完成之後，在當前目錄下，會產生4個檔案：
```
$ ls
squid-proxy.crt  squid-proxy.key  squid-server.csr  squid-server.key
```

將 `squid-proxy.crt` 與 `squid-proxy.key` 放到 container 可以讀取到的地方，例如：`/etc/squid3` ...

## 5. Restart Squid

重新啟動 Squid server 以便套用新設定
```
$ sudo docker restart squid3
```

檢查一下 port 是否有在監聽：
```
sudo netstat -tlnp | egrep '3128|3129'
```

## 6. 設定 Iptables

因為是 transparent proxy，所以還必須要加上 iptables nat rule，將 HTTP/HTTPs request 導入到 Squid 中：

```
$ sudo iptables -t nat -A PREROUTING ! -d 10.0.0.1 -p tcp --dport 80 -j REDIRECT --to 3128
$ sudo iptables -t nat -A PREROUTING ! -d 10.0.0.1 -p tcp --dport 443 -j REDIRECT --to 3129
```

這邊 `10.0.0.1` 是實體 server 的對外 IP。如果你的 NAT server 沒有提供 web service，那就可以去掉這個條件，直接變成 `iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to 3128`

或者你的 NAT server 是有區分內外網的 interface，例如假設 eth0 是連接外網、eth1 是連接內網，那就可以改成：`iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j REDIRECT --to 3128`

## 7. 用 upstart/systemd 來管理 Squid container (Options.)

如果將來 NAT server 重開機，那麼就得要自己重新啟動 container。
因此可以利用 Linux 上的 upstart(或是 systemd)來啟動 container 會方便很多。

以下用 upstart 為例( systemd 可以參考 [Docker 官方教學][4])：

1. 建立 `/etc/init/squid3.conf`，並新增以下內容：
```
description "Squid Proxy Server"
author "Maple52046"
start on filesystem and started docker
stop on runlevel [!2345]
respawn
script
	/usr/bin/docker start -a squid3
end script
```

2. 產生 `/etc/init.d/squid3`
```
sudo ln -s /lib/init/upstart-job /etc/init.d/squid3
```

3. 關閉原本的 container ，並啟動服務
```
sudo docker stop squid3
sudo service squid3 start
```
檢查一下 container 是否已經啟動
```
sudo service squid3 status
sudo docker ps | grep squid3
```

----

架設完 Squid proxy server 之後， 就可以分析 Squid 的 Log (`/var/log/squid3/access.log`)

```
1447999536.213 120894 192.168.0.3 TCP_MISS/200 7655 CONNECT 74.125.203.138:443 - HIER_DIRECT/74.125.203.138 -
1447999536.213 121078 192.168.0.3 TCP_MISS/200 4957 CONNECT 64.233.188.139:443 - HIER_DIRECT/64.233.188.139 -
1447999554.963 240143 192.168.0.3 TCP_MISS/200 5176 CONNECT 64.233.189.95:443 - HIER_DIRECT/64.233.189.95 -
1447999555.327 240351 192.168.0.3 TCP_MISS/200 5961 CONNECT 64.233.189.95:443 - HIER_DIRECT/64.233.189.95 -
1447999566.344   3061 192.168.0.3 TCP_MISS_ABORTED/000 0 GET http://secclientgw.alipay.com/product/3001/2.4.0.0/version.xml? - HIER_DIRECT/110.76.20.11 -
1447999567.267 112594 192.168.0.3 TCP_MISS/200 55546 CONNECT 23.48.140.135:443 - HIER_DIRECT/23.48.140.135 -
1447999574.672 240106 192.168.0.3 TCP_MISS/200 4895 CONNECT 74.125.204.139:443 - HIER_DIRECT/74.125.204.139 -
1447999574.975 120170 192.168.0.3 TCP_MISS/200 744 CONNECT 54.230.212.62:443 - HIER_DIRECT/54.230.212.62 -
1447999578.090 240080 192.168.0.3 TCP_MISS/200 1310 CONNECT 74.125.203.139:443 - HIER_DIRECT/74.125.203.139 -
1447999631.985  65873 192.168.0.3 TCP_MISS/200 7278 CONNECT 31.13.87.1:443 - HIER_DIRECT/31.13.87.1 -
```

Reference
=========

1. [第十七章、區網控制者： Proxy 伺服器][5] - 鳥哥的 Linux 私房菜
1. [sameersbn/squid:3.3.8-4][1]
1. [Installing Docker on Gentoo Linux][2]
1. [How to setup client for squid transparent proxy?](http://serverfault.com/questions/610232/how-to-setup-client-for-squid-transparent-proxy)
1. [Squid – SSL Certificate](http://busylog.net/squid-ssl-certificate/)
1. [拿 RAM 當硬碟來用(RAM Disk)](http://blog.longwin.com.tw/2006/01/ram_disk_build_method/)
1. [How to Enable SSL Squid](http://www.ehow.com/how_7498953_enable-ssl-squid.html)
1. [Copying files from host to docker container][3]
1. [How to create a self-signed certificate with openssl?](http://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl)
1. [Automatically start containers][4]

[1]: https://github.com/sameersbn/docker-squid
[2]: http://docs.docker.com/engine/installation/gentoolinux/
[3]: http://stackoverflow.com/questions/22907231/copying-files-from-host-to-docker-container
[4]: https://docs.docker.com/v1.8/articles/host_integration/
[5]: http://linux.vbird.org/linux_server/0420squid.php

圖片來源
=======

1. http://www.peppercan.com/wp-content/uploads/2012/03/cloud.png
2. http://orig05.deviantart.net/4a1c/f/2010/239/0/b/pc\_clipart\_by\_chiprunner-d2xfpv6.png
