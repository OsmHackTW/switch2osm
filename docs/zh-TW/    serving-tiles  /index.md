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

Serving your own maps is a fairly intensive task. Depending on the size of the area you’re interested in serving and the traffic you expect the system requirements will vary. In general, requirements will range from 10-20GB of storage, 4GB of memory, and a modern dual-core processor for a city-sized region to 1TB of fast storage, 24GB of memory, and a quad-core processor for the entire planet.

We would recommend that you begin with extracts of OpenStreetMap data – for example, a city, county or small country – rather than spending a week importing the whole world (planet.osm) and then having to restart because of a configuration mistake! You can download extracts from:

* [Geofabrik](https://download.geofabrik.de/){: target=_blank} (countries and provinces)
* [Protomaps Extracts](https://protomaps.com/extracts){: target=_blank} (minutely-updated cities and small countries)
* [download.openstreetmap.fr](https://download.openstreetmap.fr/){: target=_blank}

## 工具組合

We use a series of tools for generating and serving map tiles.

**Apache** provides the front end server that handles requests from your web browser and passes the request to mod_tile. The Apache web server can also be used to serve static web content like the HTML, JavaScript, or CSS for your map webpage.

Once Apache handles the request from the web user, it passes the request to mod_tile to deal with. Mod_tile checks if the tile has already been created and is ready for use or whether it needs to be updated due to not being in the cache already. If it is already available and doesn’t need to be rendered, then it immediately sends the tile back to the client. If it does need to be rendered, then it will add it to a “render request” queue, and when it gets to the top of the queue, a tile renderer will render it and send the tile back to the client.

We use a tool called **Mapnik** to render tiles. It pulls requests from the work queue as fast as possible, extracts data from various data sources according to the style information, and renders the tile. This tile is passed back to the client and moves on to the next item in the queue.

For rendering, OpenStreetMap data is stored in a **PostgreSQL** database created by a tool called **osm2pgsql**. These two pieces work together to allow efficient access to the OpenStreetMap geographic data. It is possible to keep the data in the PostgreSQL database up to date using a stream of diff files produced every 60 seconds on the main OpenStreetMap server.
