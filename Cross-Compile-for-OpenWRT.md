---
title: Cross-compile for OpenWRT
date: 2015-05-14 09:55:00
categories: Programming
tags:
  - Cross Compile
  - OpenWRT
  - Sniffex
aliases:
  - 2015/05/14/Cross-Compile-for-OpenWRT/
---
雖然不是很懂自己在幹嘛，不過至少是參考並修改別人的方式，完成了幾個簡單的範例。

- 編譯環境：
  - Operating System: Ubuntu 14.04 x64
  - OpenWRT version: 14.07

<!-- more -->

# Download OpenWRT source code

```
cd ~
mkdir ~/crosscompile
git clone git://git.openwrt.org/14.07/openwrt.git
```

# Build OpenWRT

```
cd openwrt
make menuconfig
```

1. Choice "Target System" and "Target Profile"
2. "Advanced configuration options (for developers)" -> "Target Options"
3. "Advanced configuration options (for developers)" -> "Toolchain Options" -> "Binutils Version" -> "Linaro binutils 2.24"

Save configuration, then:
```
make defconfg
make
```

如果嫌 `make` 不夠快，可以再加上 `-j N` 開啟其他 cpu core 來編譯。 EX： `make -j 3`(for 雙核心)。

# Generate environment configuration

- Create configuration

Create a new file:
```
cd ~/crosscompile
vi env.sh
```

Content:
```bash ~/crosscompile/env.sh
export STAGING_DIR=/home/maple/crosscompile/openwrt/staging_dir
export TOOLCHAIN_DIR=$STAGING_DIR/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2
export LDCFLAGS=$TOOLCHAIN_DIR/usr/lib
export LD_LIBRARY_PATH=$TOOLCHAIN_DIR/usr/lib
export PATH=$TOOLCHAIN_DIR/bin:$PATH
```

Change value of **STAGING_DIR** by the path of OpenWRT source code

- Source environment

```
source env.sh
```

每一次 login 都要作這個步驟，因此有兩種(懶人)方法可以簡化這個步驟

1. `echo 'source ~/crosscompile/env.sh' >> ~/.bashrc`
2. `cat ~/crosscompile/env.sh | sudo tee -a /etc/profile` or ` sudo mv ~/crosscompile/env.sh /etc/profile.d/`

之後登入就會自動讀取 env.sh 裡面的環境變數了

# Cross compile

## Compile a software package

_You need activate environment parameter before this step_

```
cd ~/crosscompile
wget -O- http://downloads.openwrt.org/sources/bluez-libs-3.36.tar.gz | tar zxf -
cd bluez-libs-3.36
./configure --prefix=$TOOLCHAIN_DIR --build=mips-openwrt-linux-gnu --host=mips-openwrt-linux-uclibc
make
make install
```

## Hello world

```
cd ~/crosscompile
mkdir helloworld
cd helloworld
```

- Generate source code

```
vi helloworld.c
```

```c helloworld.c
#include <stdio.h>

int main() {
    printf("Hello World!\n");
    return 0;
}
```

### Compile single source code

```
mips-openwrt-linux-gcc -o helloworld helloworld.c
```

### Compile with Makefile

- Generate `Makefile`

```makefile ~/crosscompile/helloworld/Makefile
INCLUDE_DIR=$(TOOLCHAIN_DIR)/usr/include

CC=mips-openwrt-linux-gcc

CFLAGS= -std=gnu99
LDFLAGS=-lbluetooth

SOURCES=helloworld.c
OBJS=$(SOURCES:.c=.o)

all: helloworld

helloworld.o: helloworld.c
	$(CC) -c $(CFLAGS) -I $(INCLUDE_DIR) -o $@ $<

%.o: %.c %.h
	$(CC) -c $(CFLAGS) -I $(INCLUDE_DIR) -o $@ $<

helloworld: $(OBJS)
	$(CC) $(LDFLAGS) $(CFLAGS) -o helloworld $(OBJS)

clean:
	rm *.o helloworld
```

縮排必須是一格 tab，不能是空白鍵，否則會產生 `Makefile:17: *** missing separator.  Stop.` 錯誤。

- Execute `make` command _(You need activate environment parameter before this step)_


完成


## Sniffex

### Install libpcap

_You need activate environment parameter before this step_

```
cd ~/crosscompile
wget -O- http://downloads.openwrt.org/sources/libpcap-1.5.3.tar.gz | tar zxf -
cd libpcap-1.5.3
./configure --prefix=$TOOLCHAIN_DIR --build=mips-openwrt-linux-gnu --host=mips-openwrt-linux-uclibc --with-pcap=null
make
make install
```

### Compile sniffex

- Install packages

```
sudo apt-get install m4 flex bison
```

- Create source code directory

```
cd ~/crosscompile
mkdir sniffex
```

- Get source code

```
wget http://www.tcpdump.org/sniffex.c
```

- Generate `Makefile`

```makefile ~/crosscompile/sniffex/Makefile
INCLUDE_DIR=$(TOOLCHAIN_DIR)/usr/include

CC=mips-openwrt-linux-gcc

CFLAGS= -Wall
LDFLAGS=-lpcap

SOURCES=sniffex.c
OBJS=$(SOURCES:.c=.o)

all: sniffex

sniffex.o: sniffex.c
	$(CC) -c $(CFLAGS) -I $(INCLUDE_DIR) -o $@ $<

%.o: %.c %.h
	$(CC) -c $(CFLAGS) -I $(INCLUDE_DIR) -o $@ $<

sniffex: $(OBJS)
	$(CC) $(LDFLAGS) $(CFLAGS) -o sniffex $(OBJS)

clean:
	rm *.o sniffex
```

- Execute `make` command _(You need source environment parameter before this step)_

Done


# Reference

- [Cross Compiling For OpenWRT On Linux](http://telecnatron.com/articles/Cross-Compiling-For-OpenWRT-On-Linux/)
- [missing separator. Stop.](http://ccchiu.pixnet.net/blog/post/28376950)
- [[Embedded] How to Cross Compile tcpdump](http://owen-hsu.blogspot.tw/2011/03/embedded-porting-tcpdump-to-arm-emedded.html)
- [如何將tcpdump移植到arm嵌入式系統](http://read-and-thinking.blogspot.tw/2009/06/tcpdumparm.html)

# 相關文章

- [Openwrt 缺少libpcap.so.xx文件 can't load library libpcap.so.1.0](http://e2dick.com/index.php/archives/17/)
