---
layout: page
title: 其他使用案例
lang: zh-TW
---

# {{ title }}

我們專注在圖磚上，但自從開放街圖 – 特別是 – 直接能取得原始資料，你可以打造任何位置或是地理應用程式。這裡有最一開始的發掘起點；[在 OpenStreetMap Wiki](http://wiki.openstreetmap.org/wiki/Frameworks){: target=_blank} 上有完整的資源。

## 常見工具

* [Osmosis](http://wiki.openstreetmap.org/wiki/Osmosis){: target=_blank} 是全方位的載入 OSM 資料到資料庫的 Java 應用程式，大多數使用 OSM 資料的應用程式都有一定程度使用 Osmosis 的情境。
* [Osmium](http://wiki.openstreetmap.org/wiki/Osmium){: target=_blank} 是具有彈性的架構，很快變得受歡迎，相比 Osmosi 提供高度客製化的選項。
* [Mapbox Studio](https://www.mapbox.com/mapbox-studio/){: target=_blank} 是一整套製作'向量圖磚'的工具，能夠在伺服器端或是客戶端渲染圖資。

## Geocoding 服務

* [Gisgraphy](https://www.gisgraphy.com){: target=_blank} 是開源的 geocoder，為正向與反向地理編碼提供 API / 網站服務，並且有自動完成、內插、地理偏差、尋找附近功能，都能以離線或是自行架設。這套件提供開放街圖匯入器，但也提供 Openadresses、Geonames 等服務。
* [Nominatim](https://nominatim.org){: target=_blank} 是開放街圖背後的 geocoding 服務 (placename ↔ lat/long)。
* [OpenCage](https://opencagedata.com/){: target=_blank} 整合進 Nominatim 以及其他開源方案提供 geocoding API。
* [OSMNames](https://osmnames.org/){: target=_blank} - 開放街圖上的地名，可以下載，排序，以及在邊界範圍以及階層式，當然可以用來 geocoding。

## 導航引擎與服務

* [OSRM](http://project-osrm.org/){: target=_blank} 是為了 OSM 資料設計的快速導航引擎。
* [Graphhopper](https://github.com/graphhopper/graphhopper/){: target=_blank} 是具有相當小記憶體用量，運作快速的 Java 導航引擎。
* [Valhalla](https://valhalla.readthedocs.io/en/latest/){: target=_blank} 是為了車輛與大眾運輸導航的 C++ 導航引擎。
* 使用 OSM 資料的公共導航 API 由 [GraphHopper](https://www.graphhopper.com/products/){: target=_blank}、[MapQuest Open](http://open.mapquestapi.com/directions/){: target=_blank} 以及 [Mapbox](https://www.mapbox.com/directions/){: target=_blank} 提供。
* 特定用途導航 API 包括 [CycleStreets cycle routing](https://www.cyclestreets.net/api/){: target=_blank} (英國以及其他地區)

## 向量地圖函式庫 (手機)

* Android 的函式庫包括 [MapLibre Android SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}、[Mapbox Android SDK](https://www.mapbox.com/android-sdk/){: target=_blank}、[mapsforge](http://mapsforge.org/){: target=_blank}、[Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}, [Skobbler Android SDK](http://developer.skobbler.com/){: target=_blank} 以及 [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}。

* iOS 函式庫包括 [MapLibre iOS SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}、[Mapbox iOS SDK](https://www.mapbox.com/ios-sdk/){: target=_blank}、[Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}、[Skobbler iOS SDK](http://developer.skobbler.com/){: target=_blank} 以及 [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}。

## 向量地圖函式庫 (網路)

* [MapLibre GL JS](https://maplibre.org/projects/maplibre-gl-js/){: target=_blank}、[Mapbox GL JS](https://www.mapbox.com/mapbox-gl-js/){: target=_blank} 以及 [Tangram](http://tangrams.github.io/tangram/){: target=_blank} 依據 OSM 渲染向量圖磚使用 WebGL 取得更好的效能。
