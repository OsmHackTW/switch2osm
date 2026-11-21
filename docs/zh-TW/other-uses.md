---
layout: page
title: 其他應用
lang: zh-TW
---

# {{ title }}

我們著重介紹圖磚，但由於只有開放街圖（OpenStreetMap）讓你直接取用原始地圖資料，你可以打造任何位置或地理相關的應用程式。以下是最常見的起點；完整清單請見[開放街圖 Wiki](http://wiki.openstreetmap.org/wiki/Frameworks){: target=_blank}。

## 常用工具

* [Osmosis](http://wiki.openstreetmap.org/wiki/Osmosis){: target=_blank} 是將 OSM 資料載入資料庫的通用 Java 應用程式。大多數 OSM 資料應用都會用到 Osmosis。
* [Osmium](http://wiki.openstreetmap.org/wiki/Osmium){: target=_blank} 是快速普及的彈性框架，提供高度可設定的 Osmosis 替代方案。
* [Mapbox Studio](https://www.mapbox.com/mapbox-studio/){: target=_blank} 是一套產生「向量圖磚」的工具，可在伺服器端或用戶端渲染。

## 地理編碼服務

* [Gisgraphy](https://www.gisgraphy.com){: target=_blank} 是開放原始碼的地理編碼器，提供正向與反向地理編碼的 API／網路服務，支援自動完成、內插、位置偏好與附近搜尋，皆可離線執行或使用託管方案。它提供開放街圖的匯入工具，也支援 Openadresses、Geonames 等資料來源。
* [Nominatim](https://nominatim.org){: target=_blank} 是開放街圖地理編碼服務（地名 ↔ 經緯度）背後的軟體。
* [OpenCage](https://opencagedata.com/){: target=_blank} 提供整合 Nominatim 與其他開放來源的地理編碼 API。
* [OSMNames](https://osmnames.org/){: target=_blank}——來自開放街圖的地名。可下載、已排序，附邊界框與階層，可直接用於地理編碼。

## 路線規劃引擎與服務

* [OSRM](http://project-osrm.org/){: target=_blank} 是專為 OSM 資料設計的快速路線規劃引擎。
* [Graphhopper](https://github.com/graphhopper/graphhopper/){: target=_blank} 是記憶體用量小的快速 Java 路線規劃引擎。
* [Valhalla](https://valhalla.readthedocs.io/en/latest/){: target=_blank} 是用於道路車輛與大眾運輸的 C++ 路線規劃引擎。
* 使用 OSM 資料的公開路線規劃 API 有 [GraphHopper](https://www.graphhopper.com/products/){: target=_blank}、[MapQuest Open](http://open.mapquestapi.com/directions/){: target=_blank} 和 [Mapbox](https://www.mapbox.com/directions/){: target=_blank}。
* 專門的路線規劃 API 包括 [CycleStreets 自行車路線規劃](https://www.cyclestreets.net/api/){: target=_blank}（英國及其他地區）

## 向量地圖函式庫（行動裝置）

* Android 函式庫包括 [MapLibre Android SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}、[Mapbox Android SDK](https://www.mapbox.com/android-sdk/){: target=_blank}、[mapsforge](http://mapsforge.org/){: target=_blank}、[Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}、[Skobbler Android SDK](http://developer.skobbler.com/){: target=_blank} 和 [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}。

* iOS 函式庫包括 [MapLibre iOS SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}、[Mapbox iOS SDK](https://www.mapbox.com/ios-sdk/){: target=_blank}、[Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}、[Skobbler iOS SDK](http://developer.skobbler.com/){: target=_blank} 和 [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}。

## 向量地圖函式庫（網頁）

* [MapLibre GL JS](https://maplibre.org/projects/maplibre-gl-js/){: target=_blank}、[Mapbox GL JS](https://www.mapbox.com/mapbox-gl-js/){: target=_blank} 和 [Tangram](http://tangrams.github.io/tangram/){: target=_blank} 使用 WebGL 渲染 OSM 向量圖磚，效能更好。
