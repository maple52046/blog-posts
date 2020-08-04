---
title: 調整 vim 註解顏色
date: 2016-06-29 12:04:20
categories: Linux
tags: vim
aliases:
  - 2016/06/29/Change-Comment-Color-in-VIM/
---

VIM 在純 terminal 的環境下, 如果你設定 `bg=light` (預設)，註解的顏色真是令人看到眼睛都快瞎了。

![VIM Color (set bg=light)](http://i.imgur.com/ZzWbavl.png)

Solution
--------

一般來說，解決的方法通常有：

- 設定 `bg=dark`

  在 `~/.vimrc` or `/etc/vimrc` 中設定:  `set bg=dark`
  這樣整個配色都會調整成：

  ![VIM Color (set bg=dark)](http://i.imgur.com/2rucoO9.png)

  但是這種配色又不合我的胃口。

- 套用別人寫好的 color scheme 或是自己寫 color scheme

  網路上已經有不少人寫好了 scheme，如果想要套用別人的 scheme，可以參考 Tsung's Blog  的[挑選 Vim 顏色(Color Scheme)][1]這篇做設定；自己編輯的話，嗯，自己 google 吧XD

  不過對於我這種懶到不行的人來說，套用別人的 scheme 還是太麻煩惹~~~

<!-- more -->

- 單純修改 comment 的顏色

  在下面第二個 [Reference][2] 中，看到直接可以設定 `hi Comment ctermfg=<< color>>`。
  這種方式對於我來說，真是一大福音阿～～。

  那麼接下來的問題就是要挑選什麼顏色了。


Select and set color
--------------------

步驟：

1. 編輯 `~/.vimrc` 或是 `/etc/vimrc`，加入 `set t_Co=256`，然後存檔離開。
2. 接下來開啟 `vim` ，然後執行：`:runtime syntax/colortest.vim`
  ![VIM Color Name](http://i.imgur.com/ttgTpic.png)
  這邊我打算挑選 **lightblue**。
3. 執行 `:q!` 離開 Vim。
4. 再次編輯 `~/.vimrc` 或是 `/etc/vimrc`，加入 `hi Comment ctermfg=lightblue`，然後存檔離開。

接下來就可以看到註解顏色的改變：

![VIM set Comment color is lightblue](http://i.imgur.com/2Xa9DpE.png)


### 進階設定

如果上述的顏色還是不滿意，那可以這樣設定：

1. 設定 Bash 環境，開啟 256 色。
```
export TERM=xterm-256color
```

2. 下載並執行 [256-xterm-colors](https://github.com/gawin/bash-colors-256/blob/master/256-xterm-colors):
```
wget https://raw.githubusercontent.com/gawin/bash-colors-256/master/256-xterm-colors
ruby 256-xterm-colors
```
  (記得要先安裝 `ruby`)。
  執行後會得到以下的顏色表：
  ![Bash 256 color](https://raw.githubusercontent.com/gawin/bash-colors-256/master/bash_256_colors_iterm_screenshot.png)

3. 選擇一個顏色，例如: `033`。然後像之前一樣設定 `~/.vimrc` 或是 `/etc/vimrc`:
```
hi Comment ctermfg=033
```

接下來就可以看到新的註解顏色:

![VIM set Comment color is 033](http://i.imgur.com/DTnSPq5.png)



Reference
---------

1. [挑選 Vim 顏色(Color Scheme)][1]
2. [CentOS vim 將看到脫窗的註解文字 深藍色 改變顏色][2]
3. [誤打誤撞研究了 Vim 的顏色設定][3]
4. [256 xterm colors][4] - Github

[1]: https://blog.longwin.com.tw/2009/03/choose-vim-color-scheme-2009/
[2]: http://shazi.info/centos-vim-%e5%b0%87%e7%9c%8b%e5%88%b0%e8%84%ab%e7%aa%97%e7%9a%84%e8%a8%bb%e8%a7%a3%e6%96%87%e5%ad%97-%e6%b7%b1%e8%97%8d%e8%89%b2-%e6%94%b9%e8%ae%8a%e9%a1%8f%e8%89%b2/
[3]: http://aknow-work.blogspot.tw/2013/05/vim-color.html
[4]: https://github.com/gawin/bash-colors-256
