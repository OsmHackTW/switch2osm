---
layout: docs
title: 使用圖磚
lang: zh-TW
---

# {{ title }}

最快一小時內就可以將現有網站轉換到開放街圖（OpenStreetMap）：只需選定 JavaScript API 和圖磚服務供應商，即可開始使用。隨著需求增加，可考慮改用專業服務供應商提供的自訂圖磚，或自行產生圖磚。

## 選擇 API／函式庫

與商業線上地圖服務不同，開放街圖並沒有提供必須使用的「官方」JavaScript 函式庫。你可以使用任何符合需求的函式庫。最受歡迎的是開放原始碼函式庫 Leaflet；另一個知名的函式庫 OpenLayers 3 也是不錯的選擇。

[開始使用 Leaflet：輕量的網頁地圖函式庫](/using-tiles/getting-started-with-leaflet.md)

[開始使用 OpenLayers：功能完整的網頁地圖函式庫](/using-tiles/getting-started-with-openlayers.md)

[開始使用 MapLibre](/using-tiles/getting-started-with-maplibre.md)

[使用 MapLibre 加入自訂資料](/using-tiles/adding-custom-data-with-maplibre.md)

## 選擇圖磚服務供應商

除非只是少量測試，否則請勿使用 OpenStreetMap.org 本身提供的圖磚。開放街圖是由志工營運的非營利組織，無法為大規模商業用途提供圖磚。你應該使用以 OSM 資料製作圖磚的第三方服務供應商，或自行產生圖磚。

### 免費服務供應商

你可以從 [Leaflet-provider](http://leaflet-extras.github.io/leaflet-providers/preview/) 專案的預覽頁面取得清單，不過其中有些並非免費（需要 API 金鑰）。

### 付費服務供應商

請參閱[清單](/providers.md)，或繼續了解如何自行產生並提供圖磚。
