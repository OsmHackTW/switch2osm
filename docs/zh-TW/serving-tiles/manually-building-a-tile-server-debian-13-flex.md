---
layout: docs
title: 手動架設圖磚伺服器（Debian 13，flex）
dist: Debian 13
dl_timestamp: "2025-08-15T12:40:00Z"
lang: zh-TW
---

# {{ title }}

!!! info ""
    本頁說明如何安裝與設定執行自有圖磚伺服器所需的全部軟體。本教學針對 [Debian Linux](https://www.debian.org/){: target=_blank} 13（trixie）搭配 OSM Carto v6.0.0 撰寫，並於 2026 年 6 月實測。

## 軟體安裝

OSM 圖磚伺服器堆疊（Stack）是一組彼此協作、共同組成圖磚伺服器的程式與函式庫。如同開放街圖的眾多面向，要達成這個目標有很多方法，幾乎每個元件都有各具優缺點的替代方案。本教學介紹的是最標準的版本，與 OpenStreetMap.org 主要圖磚伺服器所用的架構相近。

這套堆疊由 5 個主要元件組成：`mod_tile`、`renderd`、`mapnik`、`osm2pgsql` 以及 `postgresql/postgis` 資料庫。`mod_tile` 是 Apache 模組，負責提供已快取的圖磚，並判斷哪些圖磚需要重新渲染，原因可能是尚未快取，或是已經過期。`renderd` 為不同類型的請求提供優先佇列系統，用來管理並分散渲染請求帶來的負載。Mapnik 則是實際負責渲染的軟體函式庫，供 `renderd` 呼叫。

本教學是在全新的 {{ dist }} 伺服器上撰寫與測試。如果你已安裝某些軟體的其他版本（例如從舊版升級，或另外設定了 PPA 來源），可能需要做些調整。

要建置這些元件，必須先安裝各種相依套件。

Debian 預設可能未安裝 `sudo`，因此第一部分需以 `root` 登入執行：

```sh
--8<-- "docs/assets/serving-tiles/debian13-deps.txt"
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

```sh
mapnik-render --version
```

應會顯示 Mapnik 的版本號：

```sh
version 4.0.7
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
```

注意 OSM Carto 現在已改用另一種資料庫格式，詳情請參閱 OSM Carto 的 [INSTALL.md](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md)。舊格式稱為 `pgsql`，新格式則是 `flex`。

接著確認已安裝合適版本的 `carto` 編譯器。

```sh
carto -v
```

顯示的版本號應不低於：

```log
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

接著，請確認使用者 `_renderd` 能夠存取樣式表的下載位置，但它預設無權進入你的家目錄。若樣式表放在你帳號下的 `src` 內，請執行以下指令：

```sh
chmod o+rx ~
```

若不想開放家目錄權限，也可以將檔案移至其他位置，並相應修改後續指令中的檔案路徑。

下列指令會把先前下載的開放街圖資料匯入資料庫。此步驟的磁碟 I/O 負載極高，匯入全球資料可能需要數小時、數天甚至數週，視硬體而定；較小的節選資料則快上許多。你可能需要依機器的可用記憶體調整 `-C` 值。注意此步驟以使用者 `_renderd` 執行。

```sh
sudo -u _renderd \
    osm2pgsql -O flex -S ~/src/openstreetmap-carto/openstreetmap-carto-flex.lua -d gis \
    --create --slim -C 2500 --number-processes 1 \
    ~/data/azerbaijan-latest.osm.pbf
```

各選項說明如下：

`-O flex`
: 使用 `flex` 輸出，而非 `pgsql`。

`-S ~/src/openstreetmap-carto/openstreetmap-carto-flex.lua`
: 指定處理標籤所用的 Lua 腳本。可在樣式處理 OSM 標籤前先行整理，大幅簡化樣式邏輯。

`-d gis`
: 要操作的資料庫（`gis` 過去是預設值，現在必須明確指定）。

`--slim`
: 使用 `slim` 模式，匯入後仍可更新資料庫。

`-C 2500`
: 為 osm2pgsql 匯入程序分配 2.5 Gb 記憶體。記憶體較少時可改用較小的值；若匯入程序因記憶體不足而中止，請改用更小的值或範圍更小的節選資料。

`--number-processes 1`
: 使用 1 個 CPU 核心；若有更多核心，可酌量增加。

`~/data/azerbaijan-latest.osm.pbf`
: 最後一個參數是要匯入的資料檔。

指令完成時會顯示類似「Osm2pgsql took 238s overall」的訊息。

### 停用 JIT

這是 OSM Carto 作者[建議](https://github.com/openstreetmap-carto/openstreetmap-carto/blob/master/INSTALL.md#disable-jit)的做法。

```sh
sudo -u _renderd psql -d gis -c 'ALTER SYSTEM SET jit=off;' -c 'SELECT pg_reload_conf();'
```

### 建立索引

現在需要[手動套用](https://github.com/gravitystorm/openstreetmap-carto/blob/master/CHANGELOG.md#v530---2021-01-28){: target=_blank}一些額外的索引：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f indexes.sql
```

應會顯示 16 次 `CREATE INDEX`。

## 資料庫函式

有些函式需要手動載入資料庫。可以用以下指令加入：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f functions.sql
```

### 額外的資料表

OSM Carto 6.0.0 起[需要](https://github.com/openstreetmap-carto/openstreetmap-carto/blob/master/INSTALL.md#additional-database-tables)一個資料表存放白名單中的鍵／標籤值。這份清單每次發布都會更新，可隨時用以下指令加入或更新：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f common-values.sql
```

### 下載 Shapefile

地圖的大部分資料直接來自前面下載的開放街圖資料檔，但仍需要一些 shapefile 補足（例如：低縮放等級的行政邊界）。請用先前的同一個帳號下載並建立索引：

```sh
cd ~/src/openstreetmap-carto/
mkdir data
sudo chown _renderd data
sudo -u _renderd scripts/get-external-data.py
```

此步驟需下載大量資料，可能要花一段時間，期間畫面上不會有太多訊息。有些資料會直接寫入資料庫，有些則會存放到「openstreetmap-carto」下的「data」資料夾。如果這一步出錯，可能是 Natural Earth 資料的下載位置出現變動，詳情請參考[這個 issue](https://github.com/nvkelso/natural-earth-vector/issues/581#issuecomment-913988101){: target=_blank} 以及 Natural Earth 上的類似 issue。如需變更 Natural Earth 的下載位置，請修改本機上的 [`external-data.yml`](https://github.com/gravitystorm/openstreetmap-carto/blob/master/external-data.yml){: target=_blank}。

### 字型

字型需要手動安裝，方式如下：

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.py
```

由於[目前的已知問題](https://github.com/gravitystorm/openstreetmap-carto/issues/5013)，最後可能會出現錯誤，但不會影響目前任何 OSM 資料的顯示。

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

XML 檔案的位置 `/home/accountname/src/openstreetmap-carto/mapnik.xml` 需要改成你系統上的實際路徑。`[s2o]` 和 `URI=/hot/` 也可以依喜好修改。若要在同一台伺服器渲染多組圖磚，只要再加入一個類似 `[s2o]`、名稱不同且指向其他地圖樣式的區段即可。也可以改用預設 `gis` 以外的資料庫，不過這超出本文範圍。如果記憶體只有 4Gb 左右，也應該把 `num_threads` 降為 2。選用 `URI=/hot/`，是為了讓這裡產生的圖磚更容易用來取代 OpenStreetMap.org 上的 HOT 圖磚圖層。你也可以改用其他名稱，`/hot/` 只是其中一種選擇。

Mapnik 外掛資料夾的位置現在會因架構而異。你可以執行以下指令：

```sh
sudo updatedb
locate mapnik/4.0/input
```

找出它在你伺服器上的位置，然後編輯 `/etc/renderd.conf`，讓 `[mapnik]` 區段中的 `plugins_dir` 設定正確，例如：


```ini
[mapnik]
plugins_dir=/usr/lib/aarch64-linux-gnu/mapnik/4.0/input
```

如果這項設定錯誤或缺漏，渲染圖磚時可能出現的錯誤包括：

!!! warning ""
    An error occurred while loading the map layer 's2o': Could not create datasource for type: 'postgis' (no datasource plugin directories have been successfully registered) encountered during parsing of layer 'landcover-low-zoom'

其他症狀還包括圖磚無法正確渲染，以及 `renderd` 用掉的記憶體多得出乎意料。

設定好 `renderd` 如何回應圖磚渲染請求後，接著要讓 Apache 網頁伺服器把請求轉送過去。可惜近期版本的 `mod_tile` 已移除這部分設定，不過目前仍可從這裡安裝：

```sh
cd /etc/apache2/conf-available/
sudo wget https://raw.githubusercontent.com/openstreetmap/mod_tile/python-implementation/etc/apache2/renderd.conf
sudo a2enconf renderd
sudo systemctl reload apache2
```

### 確認可以看到除錯訊息

為方便觀察渲染過程與錯誤，我們建議開啟除錯訊息，但近期版本的 `mod_tile` 預設關閉此功能。請依下列步驟開啟：

```sh
sudo nano /usr/lib/systemd/system/renderd.service
```

如果 `[Service]` 底下還沒有這一行，請加入：

```ini
Environment=G_MESSAGES_DEBUG=all
```

然後執行以下指令重新載入設定：

```sh
sudo systemctl daemon-reload
sudo systemctl restart renderd
sudo systemctl restart apache2
```

### 設定 Apache

```sh
sudo mkdir /var/lib/mod_tile
sudo chown _renderd /var/lib/mod_tile
```

完成後重新啟動 `renderd`：

```sh
sudo /etc/init.d/renderd restart
```

系統日誌中應會出現 `renderd` 服務的訊息。注意近期的 Debian 版本預設不再寫入 `/var/log/syslog`，請改用以下指令即時追蹤：

```sh
sudo journalctl -ef
```

詳情請見[這裡](https://www.debian.org/releases/bookworm/amd64/release-notes/ch-information.en.html#changes-to-system-logging){: target=_blank}。

```sh
sudo /etc/init.d/apache2 restart
```

使用 `#!sh sudo journalctl -ef` 應該會看到類似這樣的一行：

```log
 Aug 15 12:16:08 h19 apachectl[14590]: [Fri Aug 15 12:16:08.655816 2025] [tile:notice] [pid 14590:tid 14590] Loading tile config s2o at /hot/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png
```

如果沒有出現，可能表示 mod_tile 的設定檔 `/etc/apache2/conf-available/renderd.conf` 沒有正確設定或啟用，請回頭檢查上方的 `wget` 步驟。

接著，在瀏覽器中開啟 `http://yourserveripaddress/index.html`（將 `yourserveripaddress` 換成伺服器的實際位址），你應該會看到「Apache2 Debian Default Page」。

!!! tip
    如果不知道伺服器分配到哪個 IP 位址，通常可以用 `ip address` 查詢。網路設定不太複雜時，就是 `127.0.0.1` 以外的那個 `inet addr`。

若使用主機商（如 VPS 或雲端主機）的伺服器，內部位址多半與外部位址不同；外部 IP 應由主機商提供，通常就是你目前連線用的位址。

請注意，上述流程只完成 `http`（連接埠 80）網站的設定。如要啟用 `https`，還需額外設定 Apache（本文不另贅述）；若使用「Let's Encrypt」申請憑證，可一併完成 HTTPS 設定。

接著，在瀏覽器中開啟：`http://yourserveripaddress/hot/0/0/0.png`

若先前修改過 `URI=/hot/`，這裡也要跟著改。你應該會看到一張小小的世界地圖。如果沒有，請檢查顯示的錯誤訊息。常見原因是權限問題，或不小心漏掉了上面某些步驟。如果還是拿不到圖磚並出現其他錯誤，請把完整輸出貼到 Pastebin，然後到 [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank} 之類的地方提問。

## 檢視圖磚

要看到圖磚，這裡偷個懶，用 HTML 檔案 `sample_leaflet.html` 顯示一張非常簡單的地圖。取得方式如下：

```sh
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo nano sample_leaflet.html
```

編輯檔案，把 IP 位址從 `127.0.0.1` 改成 `your.server.address`，讓其他電腦也能存取這台伺服器。然後開啟 `http://your.server.address/sample_leaflet.html`。

第一次顯示地圖需要一點時間。你可以放大縮小，但依伺服器速度不同，有些圖磚一開始可能會因為來不及渲染而呈現灰色。不過渲染完成後，下次就能直接使用。系統日誌中應可看到圖磚請求。

如有需要，可以把 `/etc/apache2/conf-available/renderd.conf` 中的 `ModTileMissingRequestTimeout` 從 10 秒調高到 60 秒甚至更久，延長系統回傳灰色圖磚前等待背景渲染的時間（適用於沒有舊圖磚可用時）。修改後記得執行 `#!sh sudo service renderd restart` 和 `#!sh sudo service apache2 restart`。也不妨實際測量調整其他參數的效果。

恭喜你，你已經學會了架設圖磚伺服器！接著到[使用圖磚](/using-tiles/index.md)一節，用新架好的圖磚伺服器做出一張地圖。
