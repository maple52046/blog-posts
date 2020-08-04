---
title: Compile OpenWRT x86 (KVM guest, VirtualBox)
date: 2015-05-07 11:45:00
categories: [Network, Virtualization]
tags:
  - OpenWRT
  - KVM
aliases:
  - 2015/05/07/Compile-OpenWRT-x86-KVM-guest/
---
編譯 x86 版的 OpenWRT 與之前的方式雷同，基本上差別的就在於 `make menuconfig` 的時候要作一些額外的修改。

詳細的 OpenWRT 編譯流程請參考：[Compile OpenWRT with Open vSwitch][1]，本篇不再贅述。

# 實驗環境

- OS: Ubuntu 14.04 x64
- OpenWRT version: 14.07

<!-- more -->

# 步驟

1. 安裝編譯環境, 下載 source code, 編輯 feeds.conf, 執行 `scripts/feeds`
前置步驟與[上一篇][1]一模一樣，包含如果要順便編譯 openvswitch。
> 注意：如果先前已經有編譯過了，想要使用原本的 source code。則會需要清除先前編譯產生的檔案，否則無法順利編譯。請參考[附錄](#appendix)說明

2. <a name="menuconfig" />執行 `make menuconfig` 開始進行選擇編譯選項
  - `Target system` 選擇 `x86`
  - `Subtarget` 選擇 `KVM Guest`
  - 如果需要編譯 Virtualbox 用的 VDI image，則在 `Target Images` 中選擇 `Build VirtualBox image files (VDI)`
  - `Advanced configuration options (for developers)` **不需要作任何設定**，這點與上一篇不同
  - 選擇其他想要編譯的東西，例如：Open vSwitch, tcpdump, vim, Luci, ...
  <img class="center" src="http://user-image.logdown.io/user/10779/blog/10403/post/263856/TpohOsgTRpSKk324MFTj_%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202015-05-07%20%E4%B8%8B%E5%8D%8812.19.53.png" alt="make menuconifg(OpenWRT x86)">

3. `make V=s` 進行編譯

其餘的步驟皆與[前一篇][1]雷同

# <a name="appendix" />附錄

如果先前已經有編譯過了，想要使用原本的 source code，必須要額外作一些工作

1. 在 source code 目錄下執行 `make distclean`
2. 重新執行 `scripts/feeds update -a`
  - 如果有增加 Open vSwitch repo，則 **` libatomic patch` 不需要再執行了** ；但是需要改版本的步驟則還是需要作
3. 重新執行 `scripts/feeds install -a`
4. 回到[先前的步驟](#menuconfig)繼續編譯工作

# Reference

- [Compile OpenWRT with Open vSwitch][1]

[1]: http://worldend.logdown.com/posts/256561-compile-openwrt-with-open-vswitch
