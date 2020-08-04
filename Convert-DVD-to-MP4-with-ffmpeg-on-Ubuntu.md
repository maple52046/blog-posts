---
title: Ubuntu 上使用 ffmpeg 將 DVD 轉成 mp4
date: 2016-01-23 00:14:00
categories: Linux
tags:
  - FFmpeg
  - Ubuntu
aliases:
  - 2016/01/23/Convert-DVD-to-MP4-with-ffmpeg-on-Ubuntu/
---

[FFmpeg](https://www.ffmpeg.org/) 是一套跨平臺開發原始碼的影音轉換軟體。
利用 FFmpeg 我們可以在 Linux 上將 DVD 轉換成 mp4 或是其他格式。

Ubuntu 15.04 以上可以直接以 apt-get 安裝。不過我的平臺是 Ubuntu 14.04。
因此只能通過兩種方式安裝: ppa、手動編譯。

如果要從 PPA 安裝，請參考： https://launchpad.net/~mc3man/+archive/ubuntu/trusty-media

不過我是採用編譯的方式：https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu

<!-- more -->

----

Install FFmpeg
==============

建立資料夾 `~/ffmpeg_sources` 用來編譯 ffmpeg 與其他相依性軟體

```
mkdir -p ~/ffmpeg_sources
```

由於我的實驗環境只有我一個人使用，因此編譯軟體時，我大多設定 `prefix` 為自己家目錄；
如果想要同時也提供給其他使用者，建議可以將 `prefix` 設定成 `/usr`。

另外，如果 `prefix` 設定成 `$HOME`，就要將 `$HOME/bin` 加入到 `$PATH` 裡面：

```
export PATH="$HOME:$PATH"
```

- 安裝相依性套件

```
sudo apt-get -y --force-yes install autoconf automake build-essential \
libass-dev libfreetype6-dev libsdl1.2-dev libtheora-dev libtool \
libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev \
libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev yasm libx264-dev libmp3lame-dev \
libopus-dev
```

- H.265/HEVC video encoder

```
sudo apt-get install cmake mercurial
cd ~/ffmpeg_sources
hg clone https://bitbucket.org/multicoreware/x265
cd ~/ffmpeg_sources/x265/build/linux
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME" -DENABLE_SHARED:bool=off ../../source
make
make install
```

- AAC audio encoder

```
cd ~/ffmpeg_sources
wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
tar xzvf fdk-aac.tar.gz
cd mstorsjo-fdk-aac*
autoreconf -fiv
./configure --prefix="$HOME" --disable-shared
make
make install
```

- VP8/VP9 video encoder and decoder

```
cd ~/ffmpeg_sources
wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.5.0.tar.bz2
tar xjvf libvpx-1.5.0.tar.bz2
cd libvpx-1.5.0
./configure --prefix="$HOME" --disable-examples --disable-unit-tests
make
make install
```

- Compile and install **FFmpeg**

```
cd ~/ffmpeg_sources
wget http://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
tar xjvf ffmpeg-snapshot.tar.bz2
cd ffmpeg
PKG_CONFIG_PATH="$HOME/lib/pkgconfig" ./configure \
  --prefix="$HOME" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/include" \
  --extra-ldflags="-L$HOME/lib" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libfreetype \
  --enable-libmp3lame \
  --enable-libopus \
  --enable-libtheora \
  --enable-libvorbis \
  --enable-libvpx \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nonfree
make
make install
```

編譯完之後，就可以將編譯用的臨時資料夾刪除

```
sudo rm -rf ~/ffmpeg_sources
```

----

Convert DVD to MP4
==================

接下來，先將 DVD 光碟片放入光碟機中，然後掛載起來：

```
sudo mount /dev/sr0 /mnt
```

讀取 VOB 檔進行轉換：
```
cd /mnt/
cat VTS_01_*.VOB | ffmpeg -i - -aq 100 -ac 2 -vcodec libx264 -crf 24 -threads 0 ~/VTS_01.mp4
```

利用 `cat` 加上正規表示法，將多個 VOB 檔依序丟給 ffmpeg，轉換後的影片會自動合併成一個 video 檔。
(檔案名稱與路徑請自行更換)

FFmpeg 真的是非常好用又強大呀!!!

----

Reference:

- [Compile FFmpeg on Ubuntu, Debian, or Mint](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu)
- [FFmpeg script for DVD to mp4](http://ubuntuforums.org/showthread.php?t=1564791)
