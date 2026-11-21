---
layout: docs
title: 開始使用 Leaflet
---

# {{ title }}

## 簡介

[Leaflet](http://leafletjs.com/){: target=blank} 是用來嵌入地圖的輕量 JavaScript 函式庫。它採用寬鬆的 BSD 開放原始碼授權，可放心整合到任何網站，不必擔心法律問題。原始碼可在 [GitHub](http://github.com/Leaflet/Leaflet){: target=blank} 取得。

這裡只示範一個精簡、獨立的範例，進階用法請參考官方[教學](http://leafletjs.com/examples.html){: target=blank}與[文件](http://leafletjs.com/reference.html){: target=blank}。

## 開始使用

將以下內容複製到檔案 [leaflet.html](leaflet.html){: target=_blank}，並用瀏覽器開啟：

``` html title="leaflet.html"
--8<-- "docs/zh-TW/using-tiles/leaflet.html"
```

程式碼的詳細說明請參閱官方網站。[^1]

[^1]: 快速入門：<https://leafletjs.com/examples/quick-start/>{: target=blank}

## 延伸連結

你想要……

* 使用不同的底圖？→ Leaflet 原生支援 [TMS](https://en.wikipedia.org/wiki/Tile_Map_Service{: target=blank}) 與 [WMS](https://en.wikipedia.org/wiki/Web_Map_Service){: target=blank}。Leaflet 支援的選項請參閱[說明文件](http://leafletjs.com/reference.html#tilelayer){: target=blank}。
* 加入公司所有據點？→ 將據點整理成 [GeoJSON](http://geojson.org/){: target=blank} 格式，再[加入](http://leafletjs.com/examples/geojson.html){: target=blank}地圖中。
* 使用不同的地圖投影？→ 使用 [Proj4Leaflet](https://github.com/kartena/Proj4Leaflet){: target=blank} 外掛。
