---
title: Build OpenCV 3.2 with CUDA 8.0 and Matlab on Ubuntu 16.04/14.04
date: 2017-08-02 19:14:45
categories: Linux
toc: true
tags:
  - CUDA
  - Matlab
  - OpenCV
aliases:
  - 2017/08/02/Build-OpenCV-3-2-with-CUDA-8-0-and-Matlab/
---

本文記錄如何在 Ubuntu 16.04 (14.04) 上搭建 Cuda, Matlab 與 OpenCV 的開發環境。

# Troubleshooting

先把 troubleshooting 放前面是因為本來就是為了解決第一個 error 才有了這一篇紀錄XD。

> Error 1 與 error 2 都是發生在 Matlab R2015b 已經安裝好的情況下才產生。更準確來說是**程式讀取到 Matlab 自帶的 libtiff.h** 才產生的 error。

## Error 1

某人開發的程式，在 **Matlab** 安裝完畢後，編譯時 include Matlab library 與 header 然後出現錯誤：

![Compile error](http://i.imgur.com/KGOpOlE.png)

如果編譯時不引用到 Matlab 的東西，就不會有事情。但是這道程式就是必須要同時使用 Matlab 與 OpenCV。

[根據 google 來的資料，應該是 OpenCV 編譯時沒有啟用 TIFF (?)](https://github.com/BVLC/caffe/issues/4436)。
所以解決方法就是重新編譯 OpenCV 並且啟用 TIFF。

## Error 2

重新編譯 OpenCV 時，出現錯誤:

```
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFReadDirectory@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFWriteEncodedStrip@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFIsTiled@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFOpen@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFReadEncodedStrip@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFSetField@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFWriteScanline@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFGetField@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFScanlineSize@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFNumberOfStrips@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFSetWarningHandler@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFSetErrorHandler@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFReadEncodedTile@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFReadRGBATile@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to TIFFClose@LIBTIFF_4.0' /usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference toTIFFRGBAImageOK@LIBTIFF_4.0'
/usr/local/lib/libopencv_imgcodecs.so.3.1.0: undefined reference to `TIFFReadRGBAStrip@LIBTIFF_4.0'
collect2: error: ld returned 1 exit status
tools/CMakeFiles/compute_image_mean.dir/build.make:134: recipe for target 'tools/compute_image_mean' failed
make[2]: *** [tools/compute_image_mean] Error 1
CMakeFiles/Makefile2:473: recipe for target 'tools/CMakeFiles/compute_image_mean.dir/all' failed
make[1]: *** [tools/CMakeFiles/compute_image_mean.dir/all] Error 2
Makefile:127: recipe for target 'all' failed
make: *** [all] Error 2
```

同樣根據 error 1 的參考資料：https://github.com/BVLC/caffe/issues/4436。
在 `CMakeCache.txt` 裡面發現：`WITH_TIFF=ON`, `BUILD_TIFF=OFF`。
看起來原本 `cmake` 就有偵測到系統的 libtiff，因此預設不編譯 tiff。
既然這樣，所以解決方法就是在 `cmake` 時，增加參數 `-D BUILD_TIFF=ON`。

## Error 3

OpenCV 已經編譯完成，但原本的程式重新編譯時出現:

```
/usr/bin/ld: cannot find -lippicv
```

[根據查到的資料](https://stackoverflow.com/questions/34401117/compiling-code-with-opencv-usr-bin-ld-cannot-find-lippicv)，解決方法就是在編譯 OpenCV 時，`cmake` 階段增加 `-D WITH_IPP=ON`。

----

# System environment

兩套環境：

1. Ubuntu 16.04 with Nvidia GeForce GTX 970
2. Ubuntu 14.04 with Nvidia GeForce GT 620 OEM

第一套環境是從 OS 開始安裝;
第二套是一個已經存在的環境，只是再重新編譯 OpenCV (為了解決 TIFF 問題)。

其他的系統資訊：

- OpenCV: 3.2
- CUDA: 8.0
- Matlab: R2015b (or Octave 4.2)
- Nvidia driver: **375**
- Java:
  - **Oracla Java 8** (on site 1)
  - Oracle Java 7 (on site 2)

<!-- more -->

---

# Installation

以下安裝步驟是以 site 1 為主 (Ubuntu 16.04)。

## Install Nvidia driver and Cuda

### Install Nvidia driver

```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-375
```

### Install Cuda 8.0

根據你的平台，按照[官方](https://developer.nvidia.com/cuda-downloads)的指示選擇：

![Cuda Installation](http://i.imgur.com/NxpvipW.png)

然後下載 deb 安裝檔並依照指示進行安裝：

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
sudo apt update
sudo apt install cuda
```

## Install Java (Optional)

安裝步驟：

```
sudo add-apt-repository ppa:webupd8team/java
sudo apt update
sudo apt install oracle-java8-installer ant
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
```

## Install Matlab

Matlab 是要版權的，因此這裡不再贅述。
本文是安裝 Matlab R2015b。裝完之後，將 Matlab 的環境設定到 `~/.bashrc` 中。
例如：

```
export PATH="$PATH:/usr/local/MATLAB/R2015b/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/MATLAB/R2015b/bin/glnxa64"
export C_INCLUDE_PATH="$C_INCLUDE_PATH:/usr/local/MATLAB/R2015b/extern/include"
export LIBRARY_PATH="$LIBRARY_PATH:/usr/local/MATLAB/R2015b/bin/glnxa64"
```

然後套用新的設定:

```
source ~/.bashrc
```

## Compile and Install OpenCV 3.2

### 準備編譯環境

- 安裝相依性軟體：

```
sudo apt install build-essential cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev python-dev python-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-dev libjasper-dev libdc1394-22-dev
```

- 下載原始碼：

```
cd ~
git clone https://github.com/opencv/opencv.git
cd opencv/
git checkout 3.2.0
```

### 編譯 OpenCV

```
mkdir ~/opencv/build
cd ~/opencv/build
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -DWITH_IPP=ON -BUILD_TIFF=ON ..
make -j4 1> /dev/null
```

編譯完成後，接下來要進行安裝。個人習慣是產生 deb 檔，然後利用 dpkg 安裝 opencv，好處是 package 上的管理方便 (升級/移除...等)。
不喜歡這種方式的人，可以直接下指令 `sudo make install` 就安裝完畢了


### 安裝 OpenCV (with dpkg)

使用 checkinstall 產生 deb 檔，以方便日後的管理：

_(under ~/opencv/build directory)_

```
sudo checkinstall --install=no
```

> `checkinstall` 預設在產生 deb 檔之後會順便安裝。不過，個人習慣是先加上 `--install=no`，然後在 deb 檔產生之後再手動安裝。

![Make OPENCV deb with checkinstall (圖片忘了加上 --install=no)](http://i.imgur.com/ejD4hRc.png)

一開始先輸入 package 的介紹。
接下來會跳出一個列表，輸入 0 ~ 13 的數字去修改資訊：

![Make OPENCV deb with checkinstall](http://i.imgur.com/WDTNhr1.png)

然後按下 ENTER 開始建立 deb 檔。
完成之後，在 `~/opencv/build` 會看到 deb 檔，例如: `opencv_3.2.0-1_amd64.deb` (名稱會依據你在 checkinstall 時輸入的資訊而不同)。
執行 `dpkg` 指令進行安裝:

```
sudo dpkg -i opencv_3.2.0-1_amd64.deb
```

---

<p align="center"><strong>大功告成啦!</strong></p>

---

# Reference

- [Graphics issues after/while installing Ubuntu 16.04/16.10 with NVIDIA graphics](https://askubuntu.com/questions/760934/graphics-issues-after-while-installing-ubuntu-16-04-16-10-with-nvidia-graphics) - AskUbuntu
- [CUDA Toolkit Download](https://developer.nvidia.com/cuda-downloads) - Nvidia
- [compiling code with opencv - /usr/bin/ld: cannot find -lippicv](https://stackoverflow.com/questions/34401117/compiling-code-with-opencv-usr-bin-ld-cannot-find-lippicv) - Stack Overflow
- [Installation in Linux](http://docs.opencv.org/3.2.0/d7/d9f/tutorial_linux_install.html) - OpenCV 3.2.0
- [Error in installing Caffe with OpenCV 3.1.0 on Ubuntu 16.04](https://github.com/BVLC/caffe/issues/4436) - Github
