---
title: "在 OpenWRT 上使用 aMule 下載資源"
date: 2015-09-24T00:00:00+08:00
---

雖然說電驢/電騾的資源越來越少的樣子，但是有的時候我們還是很需要有一個能用的客戶端的．考慮到電騾的下載速度那麼慢，我們自然是想這個客戶端能夠全天候運行最好了．於是，這件事還是交給路由器吧．

## 安裝 aMule

OpenWRT 上安裝 aMule 很簡單，SSH 登錄到路由器之後，直接用 opkg 安裝即可，先刷新列表：

```bash
opkg update
```

然後安裝 aMule：

```bash
opkg install amule
```

這樣子 aMule 就安裝好了．aMule 提供了 amuled 和 amuleweb 兩個主要程序，amuled 就是所謂的守護程序，而 amuleweb 提供一個網頁界面供遠程訪問．

## 配置 aMule

aMule 默認使用主目錄下 `.aMule` 目錄作爲相關文件的存放點，也就是 `/root/.aMule/`，包括各種配置文件．但是我們需要先運行一次 amuled 才會生成這些文件．

### amuled 設定

所以，先運行：

```bash
amuled
```

終端會輸入錯誤並退出．因爲我們還沒有設置 `[ExternalConnect]`．編輯 amuled 的配置文件 `/root/.aMule/amule.conf`，其中 `AcceptExternalConnections` 項設爲 1，`ECPassword` 項設爲 MD5 加密後的密碼．經 MD5 加密的密碼可以通過以下命令獲取：

```bash
echo -n "your password here" | md5sum | cut -d ' ' -f 1
``` 

### amuleweb 設定

先寫配置文件：

```bash
amuleweb --write-config --password="password" \--admin-pass="web password"
```

這裏的 **password** 是上一步 amuled 設定中未經加密的密碼，而 **web password** 是用於網頁登錄的也是未經加密的密碼．這樣就可以生成配置文件 `/root/.aMule/remote.conf` 了，該文件中使用的密碼是經過加密的，放心可靠．

## 啓動腳本

先啓動 amuled：

```bash
amuled -f &
```

然後再啓動 amuleweb：

```bash
amuleweb &
```

現在可以打開瀏覽器，輸入地址 `http://路由器IP:4711` 即可訪問 aMule 網頁版，然後開始下載資源咯．基本講完了，至於添加 KAD 網絡，添加服務器列表等內容不在本文討論之列，恕不詳敘．不過爲了方便起見，我們可以寫兩個啓動腳本．詳情參考：[amuleweb](https://gist.github.com/hubutui/94ee2272b4098919f492) 與 [amuled](https://gist.github.com/hubutui/2973b7fe9a3e9d8ff98f)．

## 改用 aMuleGUI

其實完全可以不用 amuleweb，這貨其實功能比較少，不如直接本地用一個 aMuleGUI 遠程連接到無線路由器的 aMulee，即可像平常使用 aMule 一樣的使用無線路由器上的 aMule．電腦上安裝 aMule 軟件，然後打開 aMuleGUI，輸入密碼即可登錄．這裏說的密碼上文提到的 amuled 的配置文件 `/root/.aMule/amule.conf` 裏的 `ECPassword` 的那個沒有加密過的密碼原文．

## 參考

* Arch wiki 上的 [aMule](https://wiki.archlinux.org/index.php/AMule)
