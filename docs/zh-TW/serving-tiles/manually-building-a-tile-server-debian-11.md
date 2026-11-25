---
layout: docs
title: 手動架設圖磚伺服器（Debian 11）
dist: Debian 11
dl_timestamp: "2020-11-13T21:42:03Z"
lang: zh-TW
---

# {{ title }}

!!! info ""
    本頁說明如何安裝與設定執行自有圖磚伺服器所需的全部軟體。本教學針對 [Debian Linux](https://www.debian.org/){: target=_blank} 11（bullseye）撰寫，並於 2020 年 11 月實測。

## 軟體安裝

OSM 圖磚伺服器堆疊（Stack）是一組彼此協作、共同組成圖磚伺服器的程式與函式庫。如同開放街圖的眾多面向，要達成這個目標有很多方法，幾乎每個元件都有各具優缺點的替代方案。本教學介紹的是最標準的版本，與 OpenStreetMap.org 主要圖磚伺服器所用的架構相近。

這套堆疊由 5 個主要元件組成：`mod_tile`、`renderd`、`mapnik`、`osm2pgsql` 以及 `postgresql/postgis` 資料庫。`mod_tile` 是 Apache 模組，負責提供已快取的圖磚，並判斷哪些圖磚需要重新渲染，原因可能是尚未快取，或是已經過期。`renderd` 為不同類型的請求提供優先佇列系統，用來管理並分散渲染請求帶來的負載。Mapnik 則是實際負責渲染的軟體函式庫，供 `renderd` 呼叫。

多虧 Debian 維護者將這些套件的最新版本納入 {{ dist }}，這份說明比先前的版本簡短一些。

本教學是在全新安裝的 {{ dist }} 伺服器上撰寫並測試。如果你已經安裝了某些軟體的其他版本（例如從舊版升級，或另外設定了 PPA 套件來源），可能需要自行做些調整。

要建置這些元件，必須先安裝各種相依套件。

Debian 預設沒有安裝 `sudo`，所以第一部分要以 `root` 登入執行：

```sh
--8<-- "docs/assets/serving-tiles/debian-deps.txt"
```

在仍以 root 登入的狀態下，請確認你的一般帳號能透過 `sudo` 指令取得 root 權限。將下列指令中的 `youruseraccount` 換成你的帳號名稱。請勿直接以 `root` 身分執行後續所有步驟，否則會失敗。

```sh
usermod -aG sudo youruseraccount
exit
```

執行後會切回你的一般帳號。登出後重新登入，然後執行：

```sh
sudo whoami
```

應該會回傳 `root`。

此時系統已新增幾個帳號，可以用 `tail /etc/passwd` 查看。`postgres` 用來管理存放渲染資料的資料庫；`_renderd` 則供 `renderd` 常駐程式使用，下面許多指令都要以這個使用者身分執行。

接著要建立 PostGIS 資料庫。許多程式預設資料庫名稱為 `gis`，本教學也沿用這個慣例，但並非必要。注意下方的 `_renderd` 就是執行 `renderd` 常駐程式的使用者。

```sh
sudo -u postgres -i
createuser _renderd
createdb -E UTF8 -O _renderd gis
```

仍以 `postgres` 使用者身分，在 PostgreSQL 資料庫上設定 PostGIS：

```sh
psql
```

（會進入 `postgres=#` 提示字元）

```sh
\c gis
```

（會顯示「You are now connected to database 'gis' as user 'postgres'」。）

```sql
CREATE EXTENSION postgis;
```

（會顯示 CREATE EXTENSION）

```sql
CREATE EXTENSION hstore;
```

（會顯示 CREATE EXTENSION）

```sql
ALTER TABLE geometry_columns OWNER TO _renderd;
```

（會顯示 ALTER TABLE）

```sql
ALTER TABLE spatial_ref_sys OWNER TO _renderd;
```

（會顯示 ALTER TABLE）

```sh
\q
```

（會離開 psql，回到一般的 Linux 提示字元）

```sh
exit
```

（切回執行 `sudo -u postgres -i` 前的帳號）

## Mapnik

前面已經安裝好 Mapnik，可以用以下指令確認是否安裝正確：

```py
python3
>>> import mapnik
>>>
```

如果出現第二個 `>>>` 提示字元且沒有錯誤，就表示 Python 找得到 Mapnik 函式庫。恭喜你，你已經學會了架設圖磚伺服器！用以下指令即可離開 Python：

```py
>>> quit()
```

## 樣式表設定

所需軟體都安裝好之後，接著要下載並設定樣式表。

這裡使用的是 openstreetmap.org 網站「標準」地圖的樣式。選擇它是因為文件完整，且在世界各地都能正常運作（包括地名使用非拉丁字母的地區）。不過它也有些缺點：為求全球通用而多有取捨，結構也相當複雜，若要修改並不容易。

「OpenStreetMap Carto」的官方網址是 <https://github.com/gravitystorm/openstreetmap-carto/>{: target=_blank}，安裝說明則在 <https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md>{: target=_blank}，不過本教學會涵蓋所有必要步驟。

這裡假設樣式表放在一般帳號的 `~/src` 資料夾：

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
git pull --all
git switch --detach v5.9.0
```

之所以要切到 v5.9.0，是因為它是撰文時 OpenStreetMap 採用的最後一個版本，而 OSM Carto 之後已改用另一種資料庫格式。新版本的做法請參考 OSM Carto 的 [INSTALL.md](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md)。

接著安裝合適版本的 `carto` 編譯器。

```sh
sudo npm install -g carto
carto -v
```

顯示的版本號應不低於：

```sh
1.2.0
```

然後把 carto 專案轉換成 Mapnik 看得懂的格式：

```sh
carto project.mml > mapnik.xml
```

現在 `/home/youraccountname/src/openstreetmap-carto/mapnik.xml` 已經有一份 Mapnik XML 樣式表。

## 匯入資料

一開始先只匯入少量測試資料。可以下載的地方很多，其中 `download.geofabrik.de` 的選擇相當豐富。這個範例會下載亞塞拜然（Azerbaijan）的資料，目前大約 32Mb。

前往 <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank}，記下「This file was last modified」的日期（例如「{{ dl_timestamp }}」）。之後要將資料庫更新至最新的 OSM 編輯內容時，會用到這個日期。下載方式如下：

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

下列指令會把先前下載的開放街圖資料匯入資料庫。此步驟的磁碟 I/O 負載極高，匯入全球資料可能需要數小時、數天甚至數週，視硬體而定；較小的節選資料則快上許多。你可能需要依機器的可用記憶體調整 `-C` 值。注意此步驟以使用者 `_renderd` 執行。

```sh
sudo -u _renderd \
    osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 2500 --number-processes 1 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/azerbaijan-latest.osm.pbf
```

各選項說明如下：

`-d gis`
: 要操作的資料庫（`gis` 過去是預設值，現在必須明確指定）。

`--create`
: 將資料匯入空的資料庫，而不是附加到現有資料庫。

`--slim`
: osm2pgsql 支援多種資料表結構，`slim` 適用於渲染。

`-G`
: 決定多重多邊形（multipolygon）的處理方式。

`--hstore`
: 讓沒有對應資料庫欄位的標籤也能用於渲染。

`--tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua`
: 指定處理標籤所用的 Lua 腳本。可在樣式處理 OSM 標籤前先行整理，大幅簡化樣式邏輯。

`-C 2500`
: 為 osm2pgsql 匯入程序分配 2.5 Gb 記憶體。記憶體較少時可改用較小的值；若匯入程序因記憶體不足而中止，請改用更小的值或範圍更小的節選資料。

`--number-processes 1`
: 使用 1 個 CPU 核心；若有更多核心，可酌量增加。

`-S ~/src/openstreetmap-carto/openstreetmap-carto.style`
: 依此檔案建立資料庫欄位（與 openstreetmap-carto 原始版本相同）。

`~/data/azerbaijan-latest.osm.pbf`
: 最後一個參數是要匯入的資料檔。

指令完成時會顯示類似「Osm2pgsql took 238s overall」的訊息。

### 建立索引

從 v5.3.0 版開始，有些額外的索引需要[手動套用](https://github.com/gravitystorm/openstreetmap-carto/blob/master/CHANGELOG.md#v530---2021-01-28){: target=_blank}：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f indexes.sql
```

應會顯示 16 次 `CREATE INDEX`。

## 資料庫函式

OSM Carto 5.9.0（2024 年 10 月發布）有些函式需手動載入資料庫，可隨時用以下指令新增或重新載入：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f functions.sql
```

### 下載 Shapefile

地圖的大部分資料直接來自前面下載的開放街圖資料檔，但仍需要一些 shapefile，例如低縮放等級的行政邊界。請用先前的同一個帳號下載並建立索引：

```sh
cd ~/src/openstreetmap-carto/
mkdir data
sudo chown _renderd data
sudo -u _renderd scripts/get-external-data.py
```

此步驟需下載大量資料，可能要花一段時間，期間畫面上不會有太多訊息。有些資料會直接寫入資料庫，有些則會存放到「openstreetmap-carto」下的「data」資料夾。如果這一步出錯，可能是 Natural Earth 資料的下載位置出現變動，詳情請參考[這個 issue](https://github.com/nvkelso/natural-earth-vector/issues/581#issuecomment-913988101){: target=_blank} 以及 Natural Earth 的其他 issue。如需變更 Natural Earth 的下載位置，請修改本機上的 [`external-data.yml`](https://github.com/gravitystorm/openstreetmap-carto/blob/master/external-data.yml){: target=_blank}。

### 字型

Carto v5.6.0 及以上版本需要手動安裝字型：

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

選擇亞塞拜然當測試資料區域，一方面是因為範圍小，另一方面是當地有些地名含有非拉丁字元。

## 設定網頁伺服器

### 設定 renderd

{{ dist }} 上 `renderd` 的設定檔是 `/etc/renderd.conf`。請使用 nano 等文字編輯器開啟：

```sh
sudo nano /etc/renderd.conf
```

在檔案最後加入類似以下的區段：

```ini
[s2o]
URI=/hot/
TILEDIR=/var/lib/mod_tile
XML=/home/accountname/src/openstreetmap-carto/mapnik.xml
HOST=localhost
TILESIZE=256
MAXZOOM=20
```

XML 檔案的位置 `/home/accountname/src/openstreetmap-carto/mapnik.xml` 要改成你系統上的實際路徑。你也可以視需要修改 `[s2o]` 和 `URI=/hot/`。若要在同一台伺服器渲染多組圖磚，只要再加入一個類似 `[s2o]`、名稱不同且指向其他地圖樣式的區段即可。也可以改用預設 `gis` 以外的資料庫，不過這超出本文件的範圍。如果記憶體只有 2Gb 左右，也應該把 `num_threads` 降到 2。選用 `URI=/hot/`，是為了讓這裡產生的圖磚更容易用來取代 OpenStreetMap.org 上的 HOT 圖磚圖層。你也可以改用其他名稱，`/hot/` 只是其中一種選擇。

本教學最初撰寫時，Debian 提供的 Mapnik 版本是 3.0，檔案中 `[mapnik]` 區段的 `plugins_dir` 設定為 `/usr/lib/mapnik/3.0/input`。目前已改為 3.1，因此對應的值是 `/usr/lib/mapnik/3.1/input`。日後可能還會再改變。如果渲染圖磚時出現類似以下的錯誤：

!!! warning ""
    An error occurred while loading the map layer 's2o': Could not create datasource for type: 'postgis' (no datasource plugin directories have been successfully registered) encountered during parsing of layer 'landcover-low-zoom'

請確認 `/usr/lib/mapnik` 裡有哪些版本資料夾，並檢查 `/usr/lib/mapnik/(version)/input` 中是否有 `postgis.input` 檔案。

### 設定 Apache

```sh
sudo mkdir /var/lib/mod_tile
sudo chown _renderd /var/lib/mod_tile
```

完成後重新啟動 `renderd`：

```sh
sudo /etc/init.d/renderd restart
```

查看 `/var/log/syslog` 時，應該會看到來自 `renderd` 服務的訊息。一開始會有一些字型錯誤，目前先不用理會。接著：

```sh
sudo /etc/init.d/apache2 restart
```

在 `syslog` 中應該會看到類似以下的訊息：

```log
Nov 14 14:24:55 servername apachectl[19119]: [Sat Nov 14 14:24:55.526717 2020] [tile:notice] [pid 19119:tid 140525098995008] Loading tile config s2o at /hot/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png
```

接著，在瀏覽器中開啟 `http://your.server.ip.address/index.html`（將 `your.server.ip.address` 換成伺服器的實際位址），你應該會看到「Apache2 Debian Default Page」。

!!! tip
    如果不知道伺服器分配到哪個 IP 位址，通常可以用 `ifconfig` 查詢；網路設定不太複雜時，就是 `127.0.0.1` 以外的那個 `inet addr`。

若使用主機商（如 VPS 或雲端主機）的伺服器，內部位址多半與外部位址不同；外部 IP 應由主機商提供，通常就是你目前連線用的位址。

請注意，上述流程只完成 `http`（連接埠 80）網站的設定。如要啟用 `https`，還需額外設定 Apache（本文不另贅述）；若使用「Let's Encrypt」申請憑證，可一併完成 HTTPS 設定。

接著，在瀏覽器中開啟：`http://your.server.ip.address/hot/0/0/0.png`

若先前修改過 `URI=/hot/`，這裡也要跟著改。你應該會看到一張小小的世界地圖。如果沒有，請檢查顯示的錯誤訊息。常見原因是權限問題，或不小心漏掉了上面某些步驟。如果還是拿不到圖磚並出現其他錯誤，請把完整輸出貼到 Pastebin，然後到 [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank} 之類的地方提問。

## 檢視圖磚

要看到圖磚，這裡偷個懶，用 HTML 檔案 `sample_leaflet.html` 顯示一張非常簡單的地圖。取得方式如下：

```sh
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo nano sample_leaflet.html
```

編輯檔案，把 IP 位址從 `127.0.0.1` 改成 `your.server.address`，讓其他電腦也能存取這台伺服器。然後開啟 `http://your.server.address/sample_leaflet.html`。

第一次顯示地圖需要一點時間。你可以放大縮小，但依伺服器速度不同，有些圖磚一開始可能會是灰色，因為來不及渲染並送到瀏覽器。不過渲染完成後，下次就能直接使用。`/var/log/syslog` 中應可看到圖磚請求。

如有需要，你可以把 `/etc/apache2/conf-available/renderd.conf` 中的 `ModTileMissingRequestTimeout` 設定從 10 秒提高到 30 或 60 秒，延長系統回傳灰色圖磚前等待背景渲染的時間。修改後記得執行 `#!sh sudo service renderd restart` 和 `#!sh sudo service apache2 restart`。

恭喜你，你已經學會了架設圖磚伺服器！接著到[使用圖磚](/using-tiles/index.md)一節，用新架好的圖磚伺服器做出一張地圖。
