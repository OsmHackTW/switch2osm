---
layout: docs
title: 自架圖磚服務
lang: zh-TW
---

# {{ title }}

改用第三方服務供應商的圖磚，是換到開放街圖（OpenStreetMap）最簡單的方式，費用也清楚明確。不過如果想完全自己掌控，也可以自行渲染並提供圖磚。本節說明具體做法。

![原始 OSM 資料](/assets/img/raw-osm-data.webp){ data-title="原始 OSM 資料" data-description="透過 OSM API 取得以供編輯的開放街圖原始資料，於 JOSM 中以預設地圖繪製樣式呈現。https://www.openstreetmap.org/#map=16/18.0253/-63.0485" } | ![圖磚伺服器](/assets/img/vector_tiles_pyramid_structure_window.webp){ data-title="地圖圖磚金字塔" data-description="地圖在每個縮放等級都會切割成稱為「圖磚」的小區塊，每張圖磚的尺寸通常是 256×256 像素。" } | ![地圖使用情境](/assets/img/map-usage.webp){ data-title="在你的網站上顯示圖磚" data-description="由圖磚伺服器產出的圖磚，會顯示在用戶端的網頁瀏覽器或其他應用程式中。" }
:--:|:--:|:--:
:simple-openstreetmap: :material-database-import: 開放街圖原始資料 | :material-server: :material-checkerboard-plus: 你自己的圖磚伺服器 | :fontawesome-solid-users: :octicons-browser-16: 瀏覽你網站的使用者

## 自架圖磚適合你嗎？

產生與提供圖磚都需要相當多的硬體資源；若要涵蓋全球並定期更新，需求更高。

如果你要架設自己的圖磚伺服器，我們建議使用 [Ubuntu Linux](https://ubuntu.com/){: target=_blank} 或 [Debian](https://www.debian.org/releases/){: target=_blank}。

## 架設選項

1. 直接安裝在本機系統：[Debian 13（pgsql）](manually-building-a-tile-server-debian-13.md)、[Debian 13（flex）](manually-building-a-tile-server-debian-13-flex.md)、[Debian 12](manually-building-a-tile-server-debian-12.md)、[Ubuntu 24.04](manually-building-a-tile-server-ubuntu-24-04-lts.md) 或 [Debian 11](manually-building-a-tile-server-debian-11.md)。

2. 使用 [Docker](using-a-docker-container.md)。

## 系統需求

由於自行提供地圖服務相當消耗資源，實際的系統需求會依服務區域的大小與預期流量而異。一般而言，城市規模的區域大約需要 10–20 GB 儲存空間、4 GB 記憶體與新款雙核心處理器；若要涵蓋全球，則需要 1 TB 高速儲存空間、24 GB 記憶體與四核心處理器。

我們建議先從部分區域的開放街圖資料（例如單一縣市、都會區、州郡或小型國家）著手；避免耗費一週時間匯入全球資料（planet.osm），最後卻因為設定出錯而前功盡棄！你可以從以下網站下載分區資料：

* [Geofabrik](https://download.geofabrik.de/){: target=_blank}（各國與地區）
* [slice.openstreetmap.us](https://slice.openstreetmap.us){: target=_blank}（每分鐘更新的城市與小型國家）
* [download.openstreetmap.fr](https://download.openstreetmap.fr/){: target=_blank}

## 工具組合

產生與提供地圖圖磚會用到一系列工具。

**Apache** 擔任前端網頁伺服器，負責接收來自瀏覽器的請求並轉交給 mod_tile；同時也可以用來提供靜態網頁內容，例如地圖頁面所需的 HTML、JavaScript 與 CSS。

Apache 收到網頁使用者的請求並轉交給 mod_tile 後，mod_tile 會檢查目標圖磚是否已經產生並可直接使用，或是因尚未快取而需要更新。若圖磚已就緒、無須重新渲染，就會立即傳回用戶端；若需要渲染，則把請求加入「渲染請求」佇列，輪到它時再由圖磚渲染器繪製並傳回用戶端。

圖磚由 **Mapnik** 負責渲染。它會盡快從工作佇列取出請求，依樣式設定從各個資料來源擷取資料並渲染圖磚；圖磚傳回用戶端後，再接著處理佇列中的下一個項目。

渲染所需的開放街圖資料存放在 **PostgreSQL** 資料庫中，資料庫由 **osm2pgsql** 建立。兩者搭配，讓系統能有效率地存取開放街圖的地理資料。此外，也可以利用開放街圖主伺服器每 60 秒產生的差異檔串流，讓 PostgreSQL 資料庫中的資料隨時保持最新。
