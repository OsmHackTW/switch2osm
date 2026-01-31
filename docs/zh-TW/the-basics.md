---
layout: page
title: 基本資訊
lang: zh-TW
---

# {{ title }}

## 挑戰

您目前的地圖服務商提供兩樣事情：

* 一組`圖磚` (正方形的地圖影像) 排列在一起組成地圖
* JavaScript API，或是行動裝置 app 的其他同等的函式庫來檢視地圖

要轉換使用開放街圖，您必須將上述兩樣事情取代掉。

## 圖磚

![圖磚](/assets/img/tiles.webp){ align=left .off-glb }
地圖圖磚，(通常) 256 x 256 像素的影像，由地圖資料庫繪製 (“算繪”) 出來。

如果您目前使用 Google 地圖，您正在使用 Google 的圖磚，由 Google 提供。由於開放街圖基金會為資源有限的非營利組織，您不能直接用 openstreetmap.org 取掉 (請看 [圖磚使用政策](https://operations.osmfoundation.org/policies/tiles/){: target=blank})。要取代的話，您可以做：

* 自己產生自己的圖磚，下載自由的 OSM 地圖資料庫以及自行算繪出來；

* 或是採用第三方供應商 (有些需要付費，有些是免費的)

OSM 地圖資料庫被稱為 planet.osm，整個完整資料庫與定期更新檔案都可以在 [planet.openstreetmap.org](http://planet.openstreetmap.org/){: target=blank} 取得。

Rendering your own tiles gives you complete control over their appearance. You can customise the maps to appear any way you like. Alternatively, third-party suppliers have OSM expertise and may have ready-prepared map styles that you can use.

### 點陣圖磚或是向量圖磚

Raster tiles and vector tiles are two distinct approaches to representing and serving map data. Each has its own advantages and use cases. Let's explore the differences between raster tiles and vector tiles to understand their strengths and limitations.

**點陣圖磚**

Raster tiles are essentially images or pictures of map data. They are pre-rendered at various zoom levels and stored as discrete image files. Here are some key characteristics of raster tiles:

* Raster tiles represent map data as a grid of pixels. Each tile is a static image that depicts a portion of the map at a specific zoom level.
* Raster tiles have a fixed appearance as they are generated with predefined styles. To change the map's visual representation, new tiles need to be rendered, which can be computationally intensive.
* Raster tiles can have larger file sizes compared to vector tiles because they store pixel-level details for each tile, resulting in higher storage requirements and slower download times.
* Raster tiles are well-suited for displaying complex cartographic styles, such as topographic maps or satellite imagery, where fine details are important.
* Raster tiles offer limited interactivity options, primarily limited to basic zooming and panning. Interacting with individual map features or dynamically modifying the map's appearance is challenging.

**向量圖磚**

Vector tiles, on the other hand, represent map data as a collection of geometric features, such as points, lines, and polygons. Here are the distinguishing features of vector tiles:

* Vector tiles store map data as individual geometries and attributes. These geometries can be scaled, rotated, and restyled in real-time, providing more flexibility and customization options.
* Vector tiles allow for dynamic styling and modification of map features. Styles can be changed on the fly, including colors, line widths, label placements, and other visual properties.
* Vector tiles are generally smaller compared to raster tiles. Since they store only geometric data and attributes, they require less storage space and result in faster transfer times.
* Vector tiles require less bandwidth for transfer, since only the necessary map data is sent to the client. This is particularly advantageous for mobile applications or areas with limited internet connectivity.
* Vector tiles enable rich interactivity and real-time rendering. Users can interact with individual map features, perform dynamic queries, and apply custom styles based on attributes, offering a more interactive and personalized map experience.

### 使用案例

The choice between raster tiles and vector tiles depends on the specific use case and requirements. Here are some scenarios where each type excels:

**點陣圖磚**

* Aesthetically detailed maps, such as topographic maps or satellite imagery.
* Static maps that don't require real-time interactivity or frequent updates.
* Cases where map data is relatively stable and doesn't need frequent modifications or styling changes.

**向量圖磚**

* Dynamic maps that require real-time customization and interactivity, such as user-driven styling or filtering.
* Mobile applications or areas with limited bandwidth or storage capacity.
* Maps with frequently changing data, where updates need to be reflected in real-time.

Both raster tiles and vector tiles have their merits depending on the use case. Raster tiles are suitable for detailed visualization and static map styles, while vector tiles excel in dynamic styling, interactivity, and efficient data transfer. By understanding the differences between these tile types, you can make an informed decision when choosing the most appropriate tile format for your specific mapping needs.

## API/函式庫

There is no single canonical library: you can choose whichever suits your needs best. The two most popular JavaScript libraries for displaying OSM tiles are:

* OpenLayers – 強大而且歷史悠久

* Leaflet – 輕量而且簡單易學

APIs are also available for mobile platforms, such as [Route-Me](https://github.com/route-me/route-me){: target=blank} (iOS) and [osmdroid](https://github.com/osmdroid/osmdroid){: target=blank} (Android).

## 授權條款

Unlike commercial providers’ data, OpenStreetMap is ‘open data’. The map data is available to you free-of-charge, with the freedom to copy and modify. OSM’s licence is the [Open Database Licence](http://opendatacommons.org/licenses/odbl/summary/){: target=blank}.

您必須遵循的義務為：

* 署名。You must credit OpenStreetMap with the same prominence that would be expected if you were using a commercial provider. See [OSM’s copyright guidelines](http://www.openstreetmap.org/copyright){: target=blank}.

* 相同方式分享。When you use any adapted version of OSM’s map data, or works produced with it, you must also offer that adapted database under the ODbL.
