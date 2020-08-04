---
title: Enable PXE boot on Intel NIC under RHEL
date: 2018-04-22 14:45:10
categories: Linux
tags:
  - PXE
  - RHEL
aliases:
  - 2018/04/22/Enable-PXE-boot-on-Intel-NIC-under-RHEL/
---

# Target problem

在 BIOS 中的 Boot order 中, 沒有網卡開機的選項, 甚至是找不到網卡; 進入到 OS 之後，網卡就出現了並且能正常運作。

## Environment

| Item | Value/Settings |
|:----:|:--------------:|
| OS | RHEL 7.4 |
| NIC | Intel xv520 |

# Solution

- Step 1:

從 Intel 官網中, 下載 BootUtil:

然後上傳至 RHEL 中並解壓縮：

- Step 2:

安裝 Kernel Source:

```
sudo yum install kernel-devel
```

- Step 3:

執行 `./bootutil64e` 可以先列出當前的網卡狀態：

```
Port Network Address Location Series  WOL Flash Firmware                Version
==== =============== ======== ======= === ============================= =======
  1   90E2BAB1EF88   218:00.0 10GbE   N/A FLASH Disabled
  2   90E2BAB1EF89   218:00.1 10GbE   N/A FLASH Disabled
  3   90E2BAB1F154    28:00.0 10GbE   N/A FLASH Disabled
  4   90E2BAB1F155    28:00.1 10GbE   N/A FLASH Disabled
```

> 在上述例子中, NIC 的 firmware 是設定成不能 flash, 這時候必須要先開啟 flash 功能。
> 執行 `./bootutil64e -NIC=3 -FLASHENABLE`：
> ```
> Connection to QV driver failed - please reinstall it!
>
> Intel(R) Ethernet Flash Firmware Utility
> BootUtil version 1.6.57.0
> Copyright (C) 2003-2017 Intel Corporation
>
> Enabling boot ROM on port 3...Success
>
> Reboot the system to enable the boot ROM on this port
>
> Port Network Address Location Series  WOL Flash Firmware                Version
> ==== =============== ======== ======= === ============================= =======
>   1   90E2BAB1EF88   218:00.0 10GbE   N/A FLASH Disabled
>   2   90E2BAB1EF89   218:00.1 10GbE   N/A FLASH Disabled
>   3   90E2BAB1F154    28:00.0 10GbE   N/A Reboot Required
>   4   90E2BAB1F155    28:00.1 10GbE   N/A FLASH Disabled
> ```

接下來執行 `./bootutil64e -NIC=3 -BOOTENABLE=PXE`，就可以打開網卡的 PXE 功能了：

```
Connection to QV driver failed - please reinstall it!

Intel(R) Ethernet Flash Firmware Utility
BootUtil version 1.6.57.0
Copyright (C) 2003-2017 Intel Corporation

Enabling boot ROM on port 1...Success

Reboot the system to enable the boot ROM on this port

Port Network Address Location Series  WOL Flash Firmware                Version
==== =============== ======== ======= === ============================= =======
  1   90E2BAB1EF88   218:00.0 10GbE   N/A FLASH Disabled
  2   90E2BAB1EF89   218:00.1 10GbE   N/A FLASH Disabled
  3   90E2BAB1F154    28:00.0 10GbE   N/A PXE                           2.1.40
  4   90E2BAB1F155    28:00.1 10GbE   N/A FLASH Disabled
```

完成, 並重開機驗證 PXE 是否正常啟動。
