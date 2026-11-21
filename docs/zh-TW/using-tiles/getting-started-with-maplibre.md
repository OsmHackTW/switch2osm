---
layout: docs
title: 開始使用 MapLibre GL
---

# {{ title }}

## 簡介

[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/){: target=blank} 是使用 WebGL 嵌入地圖的 TypeScript 函式庫。它採用寬鬆的 BSD 開放原始碼授權，可放心整合到任何網站，不必擔心法律問題。原始碼可在 [GitHub](https://github.com/maplibre/maplibre-gl-js/){: target=blank} 取得。

這裡只示範一個精簡、獨立的範例，進階用法請參考官方[教學](https://maplibre.org/maplibre-gl-js/docs/examples/){: target=blank}與[文件](https://maplibre.org/maplibre-gl-js/docs/API/){: target=blank}。

## 開始使用

顯示地圖需要三樣東西：資料來源、地圖樣式，以及存放這些內容的網站。我們將使用開放街圖基金會（OpenStreetMap Foundation）的 Shortbread 圖磚、VersaTiles Colorful 樣式，以及你自己營運的網站。

!!! info "網站託管"
    部分瀏覽器功能要求頁面必須來自安全的位置，也就是使用 HTTPS 的網站或你的本機電腦。以下假設你有一台「example.com」網站主機，具備靜態檔案託管功能，且熟悉如何將檔案上傳至該主機。

## 安裝軟體

我們需要 Node.js 18 或更新版本。如果你使用 Ubuntu 24.04 以上或 Debian 12 以上，可以這樣安裝：

```sh
sudo apt-get install nodejs
```

其他作業系統請參考其文件。你可以用 `node --version` 查看 Node.js 版本。若低於 18，就需要自行安裝新版 Node，其中一種方式是使用 [nvm](https://github.com/nvm-sh/nvm)。

## 建置樣式

我們將使用 VersaTiles colorful 樣式，這是一個使用 Shortbread 圖磚的基本樣式。Shortbread 向量圖磚綱要（schema）是適用於開放街圖資料的通用向量圖磚綱要。圖磚則來自開放街圖基金會的 Shortbread 圖磚服務。

!!! info ""
    向量圖磚的使用須遵守[向量圖磚使用政策](https://operations.osmfoundation.org/policies/vector/)。本頁的做法符合其規定，但向量圖磚服務沒有 SLA 或任何保證。如有這類需求，請自行架設或使用商業託管服務。

樣式需要樣式定義檔，以及存放圖示的 sprite 檔案。我們會建置一份指向自己 sprite 的樣式定義。

首先建立一個新資料夾來存放接下來產生的檔案。本文件中稱它為「style」，但你可以任意命名。我們會在這個資料夾裡建置所有需要的檔案，並放進「release」子資料夾：

```sh
mkdir ~/style
cd ~/style
mkdir release
```

建置 sprite 的過程可能很複雜，但我們不需要修改，直接使用預先建置好的版本即可：

```sh
curl -OL https://github.com/versatiles-org/versatiles-style/releases/download/v5.10.0/sprites.tar.gz
mkdir -p release/sprites
tar -C release/sprites -xzf sprites.tar.gz
```

接著建置樣式，改用剛複製的 sprite 與 OSMF 向量圖磚服務。

將以下內容複製到檔案 [build.ts](build.ts){: target=_blank}，並將「example.com」改為你提供圖磚的網址（包含你的網域名稱）：

```ts title="build.ts"
--8<-- "docs/assets/using-tiles/build.ts"
```

在同一個資料夾中安裝 TypeScript 與 VersaTiles 樣式，然後執行上面的腳本來建置樣式。

```sh
npm install tsx @versatiles/style@~5.10.0
node_modules/.bin/tsx build.ts
```

將以下內容複製到檔案 [maplibre.html](maplibre.html){: target=_blank}，並放到 release 資料夾中：

```html title="maplibre.html"
--8<-- "docs/zh-TW/using-tiles/maplibre.html"
```

## 發布樣式

將「release」資料夾的內容複製到你先前選定、用來提供檔案的位置。常見的做法是使用 scp 或 rsync 指令，或透過網頁介面上傳。

## 常見問題

### `node_modules/.bin/tsx build.ts` 無法執行

如果你的 Node.js 版本過舊，這個指令會執行失敗。請依前文所述，用 nvm 安裝最新版本即可解決。

### 網頁沒有載入任何內容

開啟瀏覽器的開發者工具並查看主控台（console）。最常見的原因是網址不正確。
