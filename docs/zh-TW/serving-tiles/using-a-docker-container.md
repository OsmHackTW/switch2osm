---
layout: docs
title: 使用 Docker 容器
lang: zh-TW
---

# {{ title }}

如果你只是想牛刀小試、一睹為快；又或者使用 Ubuntu 以外的作業系統並以 Docker 作容器化，可以試試[這個專案](https://github.com/Overv/openstreetmap-tile-server){: target=_blank}（感謝該專案的所有貢獻者）。該專案是基於[Ubuntu 22.04 手動架設說明](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md)建立的預建容器，可即裝即用。

## Docker

如果你尚未安裝 Docker，網路上有許多教學，例如[這篇](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10){: target=_blank}。

由於匯入資料庫的全球邊界資料相當龐大，因此即使只是小型的節選資料，也需要約 30GB 的硬碟空間。

## 開放街圖資料

本範例會下載尚比亞（Zambia）的資料並匯入，不過任何 OSM .pbf 檔案的步驟應大同小異。測試時我們建議先用較小的 .pbf 檔案。以執行 Docker 的非 root 使用者登入後，下載尚比亞的資料：

```sh
cd
wget https://download.geofabrik.de/africa/zambia-latest.osm.pbf
```

為資料建立 docker volume：

```sh
docker volume create osm-data
```

接著安裝並匯入資料：

```sh 
time \
    docker run \
    -v /home/renderaccount/zambia-latest.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    overv/openstreetmap-tile-server import
```

資料檔必須使用絕對路徑，不能用相對路徑。本範例的檔案位於「renderaccount」使用者的家目錄。若過程出錯，需先執行「docker volume rm osm-data」，再從上方的「docker volume create osm-data」重新開始。完成後，你應該會看到以下的訊息：

```sh
INFO:root: Import complete
```

如果你看到類似以下的訊息：

```sh
/data/region.osm.pbf: Is a directory
```

或

```sh
createuser: error: creation of new role failed: ERROR: role "renderer" already exists
```

就表示出現錯誤：先用「docker ps -a」找出失敗的容器，再用「docker rm」（後接容器 ID）刪除，然後依上述方式刪除並重新建立「osm-data」。

所需時間主要取決於本地網路速度與載入區域的大小。本範例使用的尚比亞範圍相對較小。

出錯時的錯誤訊息可能有些難懂，而且匯入流程失敗後無法接續執行。此外，新版 Docker 容器可能改用較新的 postgres，舊版建立的「osm-data」或許無法沿用，需用「docker volume rm osm-data」移除後重建。

想進一步了解容器實際做了哪些事，請參考[這個檔案](https://github.com/Overv/openstreetmap-tile-server/blob/master/Dockerfile){: target=_blank}。它的步驟與[手動架設圖磚伺服器](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md)的說明大致相同，只在圖磚 URL、內部帳號等細節上略有差異。容器基於 Ubuntu 22.04，通常不必進入操作；如有需要，可執行 `docker exec -it mycontainernumber bash`。

啟動圖磚伺服器：

```sh
docker run \
    -p 8080:80 \
    -v osm-data:/data/database \
    -d overv/openstreetmap-tile-server \
    run
```

要確認是否正常運作，請在瀏覽器開啟新的無痕視窗並前往：

`http://your.server.ip.address:8080/tile/0/0/0.png`

你應該會在瀏覽器中看到一張世界地圖。接著試試：

`http://your.server.ip.address:8080`

即可看到一張可以放大縮小的地圖。圖磚（尤其是低縮放等級）需要一小段時間才會出現。

### 更多資訊

這個 Docker 容器的功能遠比這個簡單範例多 —— 關於更新、效能調校等更多細節，請參閱 [readme](https://github.com/Overv/openstreetmap-tile-server/blob/master/README.md){: target=_blank}。

### 檢視圖磚

如果想要可自行修改的簡單「slippy map」（可拖曳縮放的地圖），可以使用 mod_tile「extra」資料夾中的 HTML 檔案「sample_leaflet.html」，位置在[這裡](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/extra/sample_leaflet.html){: target=_blank}。把檔案中 URL 的「hot」改成「tile」，然後在安裝 docker 容器的機器上用網頁瀏覽器開啟即可。如果容器裝在沒有網頁瀏覽器的伺服器上，還要編輯這個檔案，把「127.0.0.1」換成伺服器的 IP 位址，再複製到伺服器的「/var/www/html」底下。

如果想載入其他區域，只要從上方的「wget」步驟重新來過即可。可惜每次載入新資料，都必須刪除並重新建立「osm-data」。
