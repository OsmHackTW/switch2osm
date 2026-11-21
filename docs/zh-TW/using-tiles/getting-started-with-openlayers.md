---
layout: docs
title: 開始使用 OpenLayers
lang: zh-TW
---

# {{ title }}

## 簡介

[OpenLayers](http://openlayers.org/){: target=blank} 是用來嵌入地圖、功能完整的 JavaScript 函式庫。它採用寬鬆的 BSD 開放原始碼授權，可放心整合到任何網站，不必擔心法律問題。原始碼可在 [GitHub](https://github.com/openlayers/ol3/){: target=blank} 取得。

這裡只示範一個精簡、獨立的範例，進階用法請參考官方[教學](http://openlayers.org/en/latest/examples/)與 [API](http://openlayers.org/en/latest/apidoc/)。

## 開始使用

OpenLayers 文件的[快速入門](https://openlayers.org/doc/quickstart.html){: target=blank}章節詳細說明了如何用 OpenLayers 函式庫部署地圖。這需要花點功夫安裝 [node.js](https://nodejs.org/){: target=blank} 框架與函式庫，因此這裡不再贅述。

## 延伸連結

你想要……

* 使用不同的底圖？→ OpenLayers 原生支援 [TMS](https://en.wikipedia.org/wiki/Tile_Map_Service){: target=blank} 與 [WMS](https://en.wikipedia.org/wiki/Web_Map_Service){: target=blank}。支援哪些選項，請參閱 [OpenLayers 官方範例](http://openlayers.org/en/latest/examples/){: target=blank}與 [API](http://openlayers.org/en/latest/apidoc/){: target=blank}。
* 加入公司所有據點？→ 將據點整理成 [GeoJSON](http://geojson.org/){: target=blank} 格式，再[加入](http://openlayers.org/en/latest/examples/select-features.html){: target=blank}地圖中。
* 使用不同的地圖投影？→ 只要引入 [proj4js](http://proj4js.org/){: target=blank} JavaScript 函式庫，OpenLayers 就支援所有 Proj4 投影。此外也支援[用戶端點陣重投影](http://openlayers.org/en/latest/examples/reprojection-by-code.html){: target=blank}，因此你可以在當地投影中使用開放街圖圖磚。
  