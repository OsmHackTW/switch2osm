---
layout: docs
title: 自建圖磚伺服器
lang: zh-TW
---

# {{ title }}

使用第三方服務商的圖磚是最簡單切換使用開放街圖的方式，而且會有折扣。然而，如果想要完全控制權的話，您需要渲染以及架設您的圖磚這一章節會解釋怎麼做。

![原始 osm 資料](/assets/img/raw-osm-data.webp){ data-title="原始 OSM 資料" data-description="透過 OSM API 來編輯，在 JOSM 的預設地圖繪製樣式載入原始開放街圖資料。https://www.openstreetmap.org/#map=16/18.0253/-63.0485" } | ![圖磚伺服器](/assets/img/vector_tiles_pyramid_structure_window.webp){ data-title="地圖標題金字塔" data-description="地圖的每一層級都被切成小塊叫做圖磚。通常一塊圖磚大小是 256×256 像素。" } | ![地圖使用](/assets/img/map-usage.webp){ data-title="您的網站使用的圖磚" data-description="由您的圖磚伺服器提供的圖磚會顯示在客戶端的瀏覽器或是其他應用程式。" }
:--:|:--:|:--:
:simple-openstreetmap: :material-database-import: 開放街圖原始資料 | :material-server: :material-checkerboard-plus: 您自己的圖磚伺服器 | :fontawesome-solid-users: :octicons-browser-16: 使用者瀏覽您的網站

## 你真得需要嗎？

自己產生以及提供圖磚兩者都需要一定的硬體需求，特別是如果要包括全球範圍以及時時更新。

如果您需要設定自己的圖磚伺服器，我們建議您使用 [Ubuntu Linux](https://ubuntu.com/){: target=_blank} 或是 [Debian](https://www.debian.org/releases/){: target=_blank}。

## 選項

1. 在 [Debian 12](manually-building-a-tile-server-debian-12.md)，[Ubuntu 24.04](manually-building-a-tile-server-ubuntu-24-04-lts.md)，或是 [Debian 11](manually-building-a-tile-server-debian-11.md) 本地端安裝。

2. 使用 [docker](using-a-docker-container.md)。

## 系統需求

自架您的地圖也許有一定的難度，需視您所想要呈現的區域大小以及預計的流量，系統需求會變動很大。一般來說，一個城市區域，儲存空間需要 10-20GB，4GB 的記憶體，以及現代的雙核心處理器，而一整個地球則需要 1TB 的快存，24GB 的記憶體，以及四核心處理器。

我們建議從截取開放街圖資料起頭 - 例如說，國土小的國家，國家，城市 - 而不是花一星期時間匯入整個地球的資料 (planet.som)，接著遇到設定錯誤而需要重新開機！您可以從這些地方下載截取：

* [Geofabrik](https://download.geofabrik.de/){: target=_blank} (國家與省份)
* [Protomaps Extracts](https://protomaps.com/extracts){: target=_blank} (分鐘更新的城市以及小國家)
* [download.openstreetmap.fr](https://download.openstreetmap.fr/){: target=_blank}

## 工具組合

我們使用一系列的工具來產生與供應地圖圖磚。

**Apache** 提供前端伺服器，能夠處理來自您的灠覽器的請求，並且傳遞給 mod_tile。Apache 網頁伺服器也能夠供應靜態網頁內容 HTML、JavaScription 或是 CSS 到您的地圖網頁。

一旦 Apache 處理來自網路使用者的請求，會傳遞需求到 mod_tile 來處理。Mod_tile 會檢查圖磚是否已經生成，是否仍可用，或者是沒有在快苡而需要更新。如果已經有圖磚的話，並不需要重新渲染，接著就會回傳圖磚給客戶端。如果需要重新渲染，接著就會加到“渲染請求”佇列，當達到佇列的頂端時，圖磚渲染器會渲染出圖磚，並且回傳給客戶端。

我們使用叫做 **Mapnik** 的渲染圖磚。要渲染圖磚，需要盡快從工作佇列拉取請求，依據樣式資訊從不同資料來源截取資料，以及渲染圖磚。這些圖磚會傳遞到客戶端，接著移動到佇列中的下一項目。

要開始渲染，開放街圖資料會儲存在由 **osm2pgsql** 工具製作的 **PostgreSQL** 資料庫。這兩樣程式共同合作，允許有效率的連結開放街圖地理資料，然後也有有辦法運用每 60 秒源源不斷從主要的開放街圖伺服器產生的 dff 檔案，保持 PostgreSQL 資料庫內容在最新狀態。
