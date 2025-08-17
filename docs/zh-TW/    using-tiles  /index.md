---
layout: docs
title: 使用圖磚
lang: zh-TW
---

# {{ title }}

You can switch a website to OpenStreetMap in under an hour. Choose a JavaScript API and a tile provider, and you’re ready to go. Then, as your needs increase, you can consider custom tiles, either from a specialist provider or generated yourself.

## 選擇 API/函式庫

Unlike commercial online map providers, OpenStreetMap does not provide an “official” JavaScript library which you are required to use. Rather, you can use any library that meets your needs. The most popular is Leaflet, an open-source library. OpenLayers 3, another well know library, can also be a good fit.

[Getting started with Leaflet – a light web maps library](/using-tiles/getting-started-with-leaflet.md)

[Getting started with Openlayers – a full feature library for web maps](/using-tiles/getting-started-with-openlayers.md)

## 選擇圖磚服務商

Apart from very limited testing purposes, you should not use the tiles supplied by OpenStreetMap.org itself. OpenStreetMap is a volunteer-run non-profit body and cannot supply tiles for large-scale commercial use. Rather, you should use a third party provider that makes tiles from OSM data, or generate your own.

### 免費服務商

You can get a list using the project [Leaflet-provider](http://leaflet-extras.github.io/leaflet-providers/preview/) preview although some of them are not free (require an API key).

### 付費服務商

See [list](/providers.md). Or go on to find out how to generate and serve your own tiles.
