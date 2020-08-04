---
title: MuJoco and mujoco_py Installation
date: 2020-04-25 19:20:05
categories: Python
tags: [Python]
aliases:
  - 2020/04/25/Mujoco-Installation/
---

安裝步驟：

1. 建立目錄: `mkdir -p ~/.mujoco`
2. 從 email 中下載授權認證，並存到 `~/.mujoco/mjkey.txt`
3. 下載安裝 MoJuCo:

  ```bash
  wget https://www.roboti.us/download/mujoco200_linux.zip
  unzip mujoco200_linux.zip
  mv mujoco200_linux ~/.mojuco/mojuco200
  ```

4. 設定 bash 環境

  ```bash
  echo 'export LD_LIBRARY_PATH=$HOME/.mujoco/mujoco200/bin:$LD_LIBRARY_PATH' >> ~/.bashrc
  source ~/.bashrc
  ```

  > MuJoCo 安裝是步驟 1 ~ 4，接下來是 mujoco_py 安裝

5. 安装相依性

  - Debian/Ubuntu:

    ```bash
    sudo apt-get install gcc patchelf libglu1-mesa-dev mesa-common-dev
    ```

  - CentOS:

    ```bash
    sudo yum install gcc mesa*
    ```

6. 安裝 `mujoco_py`:

  ```bash
  pip3 install -U 'mujoco-py<2.1,>=2.0'
  ```

安裝完成

## Reference

1. [Mujoco-py](https://github.com/openai/mujoco-py) - Github
1. [ERROR: Could not build wheels for mujoco-py which use PEP 517 and cannot be installed directly](https://github.com/openai/mujoco-py/issues/492)
