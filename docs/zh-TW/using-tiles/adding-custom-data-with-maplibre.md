---
layout: docs
title: 使用 MapLibre GL 加入自訂資料
---

# {{ title }}

## 簡介

顯示地圖時，常常需要呈現自訂資料。這可能是你想在頁面上強調的資料、需要比一般底圖更詳細的資料，或是底圖中沒有的資料。本教學說明如何加入比一般底圖更詳細的內容。

## 建立含有地圖的頁面

請先依照[開始使用 MapLibre](/using-tiles/getting-started-with-maplibre.md)的步驟操作。完成後就有了起點：含有建置腳本的 `style` 資料夾，以及可以複製到網站主機的 `release` 資料夾。

!!! info "網站託管"
    部分瀏覽器功能只有在頁面來自安全的位置時才能運作，例如使用 HTTPS 的網站或你的電腦。以下假設你有一台「example.com」網站主機，具備靜態檔案託管功能，且熟悉如何將檔案上傳至該主機。

## 建置自訂資料

為了在地圖上加入更多資訊，我們先建立一個檔案，包含所有想顯示的新圖徵。本範例將使用開放街圖中的樹木與森林資料。

## 下載資料

一開始我們只載入少量測試資料。可以下載的地方很多，其中 `download.geofabrik.de` 的選擇相當豐富。本範例會下載英格蘭的資料，目前大約 1.5 GB。

```sh
cd ~/style
mkdir -p data
cd data
wget https://download.geofabrik.de/europe/united-kingdom/england-latest.osm.pbf
```

## 安裝 Tilemaker

[Tilemaker](https://tilemaker.org/) 是從開放街圖資料自行製作向量圖磚的軟體。執行方式有很多種，這裡依照其[入門指南](https://github.com/systemed/tilemaker?tab=readme-ov-file#getting-started)在 Docker 中執行。

!!! warning "從原始碼建置"
    另一種安裝 Tilemaker 的方式是從原始碼建置。這部分有[完整的文件](https://github.com/systemed/tilemaker/blob/master/docs/INSTALL.md)，但要讓它順利執行可能比 Docker 麻煩。

### 安裝 Docker

如果你還沒安裝 Docker，網路上有很多教學，例如[這篇](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10){: target=_blank}。如果你使用 Debian 13 或 Ubuntu 22.04 以上版本，可以這樣安裝：

```sh
sudo apt install docker.io
```

### 在 Docker 中執行 Tilemaker

用以下指令測試 Tilemaker 能否正常執行：

```sh
docker run -it --rm -v ~/style:/style ghcr.io/systemed/tilemaker:master --help
```

執行後會列出 tilemaker 的選項說明。

## 撰寫 Tilemaker 設定

Tilemaker 設定檔用來指定要建立哪些向量圖層，以及各圖層要放入哪些資料。更複雜的範例請參閱 [tilemaker 的文件](https://github.com/systemed/tilemaker/blob/master/docs/CONFIGURATION.md)。

在本例中，我們要定義三個圖層：
1. 第一個圖層是對應單棵樹木的點，
2. 第二個圖層是代表樹列的線，以及
3. 第三個圖層是林地範圍。

將下方的設定檔另存為 `~/style/` 資料夾中的 `trees.json`：

```json title="trees.json"
--8<-- "docs/assets/using-tiles/trees.json"
```

設定檔定義了圖層名稱與產生的縮放等級，圖層內容則交由處理檔（process file）定義。

### 處理檔

接著要撰寫一個 Lua 處理檔，定義如何檢查 OSM 物件，並將它們放入剛才定義的向量圖層。

在 Tilemaker 中可以預先宣告需要的鍵（Key），加快處理速度。Tilemaker 只會處理符合這些標籤或鍵的 OSM 物件。

```lua
node_keys = { "natural=tree" }
way_keys = { "natural=tree_row", "natural=wood", "landuse=forest" }
```

接著要宣告 `node_function`，每個符合 `node_keys` 的節點都會呼叫這個函式。它需要檢查是否有 natural=tree 標籤，若有，就在 `tree_points` 圖層加入一個點。我們也會一併加入樹木的名稱與葉型。

```lua
function node_function(node)
    -- 節點上的 natural=tree 代表一棵樹
    if Find("natural") == "tree" then
        -- 將目前處理的節點加入為點
        Layer("tree_points", false)
        -- 將 name 屬性設為 name 標籤的值
        Attribute("name", Find("name"))
        -- 將 leaf_type 屬性設為 leaf_type 標籤的值
        Attribute("leaf_type", Find("leaf_type"))
    end
end
```

樹列與林地都位於路徑（way）上，因此需要撰寫一個能同時處理兩者的 `way_function`。當路徑有 `natural=tree_row` 標籤時，就把線加入線圖層；有表示林地的標籤時，則把面加入面圖層。`Layer` 函式的第二個參數決定路徑要以線或面的形式加入。

```lua
function way_function(node)
    -- 將 natural 標籤的值存入區域變數，因為會多次使用
    local natural = Find("natural")
    -- 路徑上的 natural=tree_row 代表一列樹
    if natural == "tree_row" then
        -- 將路徑加入為線
        Layer("tree_lines", false)
        -- 將 leaf_type 屬性設為 leaf_type 標籤的值
        Attribute("leaf_type", Find("leaf_type"))
    end
    if natural == "wood" or Find("landuse") == "forest" then
        -- 將路徑加入為多邊形
        Layer("tree_areas", true)
        -- 將 leaf_type 屬性設為 leaf_type 標籤的值
        Attribute("leaf_type", Find("leaf_type"))
    end
end
```

將以上內容合併後，另存為 `~/style/` 資料夾中的 `trees.lua`：


```lua title="trees.lua"
--8<-- "docs/assets/using-tiles/trees.lua"
```

## 建置圖磚

現在用 tilemaker 建置圖磚：

```sh
cd ~/style
docker run -it --rm -v ~/style:/style ghcr.io/systemed/tilemaker:master --config /style/trees.json --process /style/trees.lua --input /style/data/england-latest.osm.pbf --output /style/release/trees.pmtiles
```

這會在 `release/` 中建立一個包含樹木圖磚的 pmtiles 封存檔。

## 修改樣式

要顯示這些新資料，必須編輯樣式。MapLibre 樣式由中繼資料、來源清單與圖層清單組成。每個圖層都有 ID、指定使用的來源，並包含樣式設定。由於[MapLibre 樣式的完整指南](https://maplibre.org/maplibre-style-spec/)超出本教學範圍，這裡只做幾項簡單的修改。

我們會用文字編輯器修改 `style.json` 檔案。

第一步是加入新的來源。在 `sources` 清單中加入以下物件。

```json
"trees": {
  "type": "vector",
  "url": "pmtiles://https://example.com/trees.pmtiles"
}
```

接著要加入森林。先找到 `"id": "land-forest"` 的圖層。由於樹木改用我們自己的來源，要在 `style.json` 中移除這個圖層並加以取代。

原本的圖層是：

```json
{
    "source": "versatiles-shortbread",
    "id": "land-forest",
    "type": "fill",
    "source-layer": "land",
    "filter": [
    "all",
    [
        "in",
        "kind",
        "forest"
    ]
    ],
    "paint": {
    "fill-color": "rgb(102,170,68)",
    "fill-opacity": {
        "stops": [
        [
            7,
            0
        ],
        [
            8,
            0.1
        ]
        ]
    }
    }
}
```

將它替換為以下圖層：

```json
{
    "source": "trees",
    "id": "tree_areas",
    "type": "fill",
    "source-layer": "tree_areas",
    "paint": {
    "fill-color": "rgb(102,170,68, 0.5)"
    }
},
{
    "source": "trees",
    "id": "tree_lines",
    "type": "line",
    "source-layer": "tree_lines",
    "paint": {
    "line-color": "rgb(102,170,68, 0.5)",
    "line-width": 4
    }
},
{
    "source": "trees",
    "id": "tree_points",
    "type": "circle",
    "source-layer": "tree_points",
    "paint": {
    "circle-color": "rgb(102,170,68, 0.5)",
    "circle-radius": 4
    }
},
{
    "source": "trees",
    "id": "tree_points-trunk",
    "type": "circle",
    "source-layer": "tree_points",
    "paint": {
    "circle-color": "rgb(145, 108, 81, 0.8)",
    "circle-radius": 1.7
    }
}
```

## 顯示結果

要顯示結果，我們需要一個載入 MapLibre 與 PMTiles 外掛來呈現地圖的 HTML 檔案。這個檔案是以[開始使用 MapLibre](/using-tiles/getting-started-with-maplibre.md)指南為基礎。

```html title="maplibre.html"
--8<-- "docs/assets/using-tiles/maplibre-trees.html"
```

## 常見問題

### HTTP Range 請求

網站主機必須支援 HTTP Range 請求（HTTP Range Requests）。常見的專用 HTTP 伺服器都支援，但某些用來在本機電腦提供檔案的工具可能不支援。
