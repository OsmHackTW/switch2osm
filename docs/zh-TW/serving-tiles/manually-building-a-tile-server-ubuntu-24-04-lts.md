---
layout: docs
title: 手動架設圖磚伺服器（Ubuntu 24.04）
dist: Ubuntu 24.04
dl_timestamp: "2024-04-24T20:21:40Z"
lang: zh-TW
---

# {{ title }}

!!! info ""
    本頁說明如何安裝與設定執行自有圖磚伺服器所需的全部軟體。這份逐步教學以 [Ubuntu Linux](https://en.wikipedia.org/wiki/Ubuntu){: target=_blank} [24.04](http://www.releases.ubuntu.com/24.04/){: target=_blank}（Noble Numbat）撰寫，並於 2024 年 4 月測試。

## 軟體安裝

OSM 圖磚伺服器堆疊是一組協同運作、用來建立圖磚伺服器的程式與函式庫。和開放街圖（OpenStreetMap）的許多事情一樣，要達成這個目標有很多種方法，而且幾乎每個元件都有各具優缺點的替代方案。本教學介紹的是最標準的做法，與 OpenStreetMap.org 主要圖磚伺服器所使用的架構相近。

這套堆疊由 5 個主要元件組成：mod_tile、renderd、mapnik、osm2pgsql，以及 postgresql/postgis 資料庫。Mod_tile 是 Apache 模組，負責提供已快取的圖磚，並判斷哪些圖磚需要重新渲染，原因可能是尚未快取，或是已經過期。Renderd 為不同類型的請求提供優先佇列系統，用來管理並分散渲染請求帶來的負載。Mapnik 則是實際負責渲染的軟體函式庫，供 renderd 呼叫。

多虧 Debian 與 Ubuntu 維護者將這些套件的最新版本納入 Ubuntu 24.04，這份教學比部分舊版簡短一些。

本教學是在全新安裝的 Ubuntu 24.04 伺服器上撰寫並測試。如果你已經安裝了某些軟體的其他版本（例如從舊版升級，或另外設定了 PPA 來源），可能需要自行做些調整。

要建置這些元件，必須先安裝各種相依套件。

本教學假設你以非 root 使用者搭配「sudo」執行所有指令。不要直接以 root 身分執行以下步驟，那樣行不通。

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-24-04-deps.txt"
```

此時系統已新增幾個帳號，可以用「tail /etc/passwd」查看。「postgres」用來管理存放渲染資料的資料庫；「_renderd」則供 renderd 常駐程式使用，下面許多指令都要以這個使用者身分執行。

接著要建立 postgis 資料庫。許多程式的預設值都假設資料庫名稱為 gis，本教學也沿用這個慣例，不過這不是必要的。注意下方的「_renderd」就是 renderd 常駐程式執行時所用的使用者。

```sh
sudo -u postgres -i
createuser _renderd
createdb -E UTF8 -O _renderd gis
```

仍以 `postgres` 使用者身分，在 PostgreSQL 資料庫上設定 PostGIS：

```sh
psql
```

（這會進入 `postgres=#` 提示字元）

```sh
\c gis
```

（會回應「You are now connected to database 'gis' as user 'postgres'」。）

```sql
CREATE EXTENSION postgis;
```

（會回應 CREATE EXTENSION）

```sql
CREATE EXTENSION hstore;
```

（會回應 CREATE EXTENSION）

```sql
ALTER TABLE geometry_columns OWNER TO _renderd;
```

（會回應 ALTER TABLE）

```sql
ALTER TABLE spatial_ref_sys OWNER TO _renderd;
```

（會回應 ALTER TABLE）

```sh
\q
```

（這會離開 psql，回到一般的 Linux 提示字元）

```sh
exit
```

（回到執行上面 `sudo -u postgres -i` 之前的使用者身分）

## Mapnik

前面已經安裝好 Mapnik，可以用以下指令確認是否安裝正確：

```py
python3
>>> import mapnik
>>>
```

如果 Python 回應第二個 `>>>` 提示字元且沒有錯誤，就表示 Python 找得到 Mapnik 函式庫。恭喜！用以下指令即可離開 Python：

```py
>>> quit()
```

## 樣式表設定

所有必要軟體都安裝好之後，接下來要下載並設定樣式表。

這裡使用的樣式就是 openstreetmap.org 網站「標準」地圖所用的樣式。選擇它是因為文件完整，而且在世界各地都能正常運作（包括地名使用非拉丁字母的地方）。不過它也有些缺點：為了全球通用，設計上做了許多妥協；結構也相當複雜，如果需要修改，不容易理解和調整。

「OpenStreetMap Carto」的官方網址是 https://github.com/gravitystorm/openstreetmap-carto/ ，也有自己的安裝說明：https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md ，不過本教學會涵蓋所有必要步驟。

這裡假設樣式表放在你所用非 root 帳號的家目錄下「src」資料夾中；稍後會調整權限，讓「_renderd」使用者也能存取。

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
git pull --all
git switch --detach v5.9.0
```

需要執行「git switch」，是因為這是目前在開放街圖上看得到的最新發行版本，而 OSM Carto 正在改用另一種資料庫格式。新版本的做法請參考 OSM Carto 的 [INSTALL.md](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md)。

接著安裝合適版本的 `carto` 編譯器。

```sh
sudo npm install -g carto
carto -v
```

應該會顯示不低於以下的版本號：

```sh
1.2.0
```

然後把 carto 專案轉換成 Mapnik 能理解的格式：

```sh
carto project.mml > mapnik.xml
```

現在 `/home/youruseraccount/src/openstreetmap-carto/mapnik.xml` 已經有一份 Mapnik XML 樣式表。

## 匯入資料

一開始先只匯入少量測試資料。可以下載的地方很多，其中「download.geofabrik.de」的選擇相當豐富。這個範例會下載亞塞拜然的資料，目前約 32Mb。

瀏覽 <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank}，記下「This file was last modified」的日期（例如「2024-04-24T20:21:40Z」）。之後如果要把其他人對開放街圖的後續修改更新進資料庫，就會用到這個日期。下載方式如下：

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

接下來要確認「_renderd」使用者能存取樣式表。它必須能存取樣式表的下載位置，但預設沒有權限進入你的家目錄。如果樣式表放在你帳號底下的「src」中，那麼

```sh
chmod o+rx ~
```

就可以了。如果你不想這樣做，也可以把檔案移到別處，並修改後續指令中的檔案路徑。

以下指令會把先前下載的開放街圖資料匯入資料庫。這個步驟對硬碟 I/O 的負擔很重；依硬體不同，匯入整個地球的資料可能需要好幾個小時、好幾天，甚至好幾週。較小的節選資料匯入起來會快很多，你也可能需要試試不同的 -C 值，配合機器可用的記憶體。注意這個步驟是以「_renderd」使用者執行。


```sh
sudo -u _renderd \
    osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 2500 --number-processes 1 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/azerbaijan-latest.osm.pbf
```

以下稍微說明這些選項的意義：

`-d gis`
: 要操作的資料庫（`gis` 過去是預設值，現在必須明確指定）。

`--create`
: 將資料匯入空的資料庫，而不是附加到現有資料庫。

`--slim`
: osm2pgsql 可以使用不同的資料表結構；「slim」資料表適用於渲染。

`-G`
: 決定多重多邊形（multipolygon）的處理方式。

`--hstore`
: 讓沒有對應資料庫欄位的標籤也能用於渲染。

`--tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua`
: 指定用來處理標籤的 Lua 腳本。這是在樣式處理 OSM 標籤之前先行處理標籤的簡便方法，有機會讓樣式邏輯簡化許多。

`-C 2500`
: 為 osm2pgsql 匯入程序分配 2.5 Gb 記憶體。如果記憶體較少，可以試試較小的數值；如果匯入程序因記憶體不足而被終止，就需要改用更小的數值或更小的 OSM 節選資料。

`--number-processes 1`
: 使用 1 顆 CPU。如果有更多核心可用，可以多用一些。

`-S ~/src/openstreetmap-carto/openstreetmap-carto.style`
: 依照這個檔案建立資料庫欄位（內容其實與「openstreetmap-carto」原本的相同，沒有更動）

`~/data/azerbaijan-latest.osm.pbf`
: 最後一個參數是要匯入的資料檔。

這個指令完成時會顯示類似「osm2pgsql took 163s (2m 43s) overall」的訊息。

### 建立索引

自 v5.3.0 版起，有些額外的索引需要[手動套用](https://github.com/gravitystorm/openstreetmap-carto/blob/master/CHANGELOG.md#v530---2021-01-28){: target=_blank}：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f indexes.sql
```

應該會回應 16 次 `CREATE INDEX`。

## 資料庫函式

「OSM Carto」5.9.0 版（2024 年 10 月發布）有些函式需要手動載入資料庫，隨時都可以用以下指令新增或重新載入：

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f functions.sql
```

### 下載 Shapefile

雖然建立地圖所用的資料大多直接來自上面下載的開放街圖資料檔，但仍需要一些 shapefile，例如低縮放等級的國界。以先前使用的同一個帳號下載並建立索引：

```sh
cd ~/src/openstreetmap-carto/
mkdir data
sudo chown _renderd data
sudo -u _renderd scripts/get-external-data.py
```

這個過程需要下載相當大量的資料，可能要花一些時間，執行期間畫面上不會顯示太多內容。有些資料會直接寫入資料庫，有些則會存放在「openstreetmap-carto」底下的「data」資料夾。如果這一步出錯，可能是 Natural Earth 資料的下載位置出現變動，詳情請參考[這個 issue](https://github.com/nvkelso/natural-earth-vector/issues/581#issuecomment-913988101){: target=_blank} 以及 Natural Earth 的其他 issue。如果需要變更 Natural Earth 的下載位置，請修改你本機的那份[這個檔案](https://github.com/gravitystorm/openstreetmap-carto/blob/master/external-data.yml){: target=_blank}。

這部分的流程不太能處理意外狀況。如果你認為下載的外部資料有問題，請刪除「data」資料夾，再以正確的擁有者重新建立。如果問題仍然存在，請到[開放街圖論壇](https://community.openstreetmap.org/tags/c/help-and-support/7/all/switch2osm)提問。

### 字型

Carto v5.6.0 及以上版本需要手動安裝字型：

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

取得最後一個字型（「Noto_Emoji」）時可能會出現錯誤。這是[已知錯誤](https://github.com/gravitystorm/openstreetmap-carto/issues/5013)，可以忽略；目前還在等人接受某個 [pull request](https://github.com/gravitystorm/openstreetmap-carto/pull/4893)，或提出其他解法。

選擇亞塞拜然當測試資料區域，一方面是因為範圍小，另一方面是當地有些地名含有非拉丁字元。

## 設定網頁伺服器

### 設定 renderd

{{ dist }} 上 `renderd` 的設定檔是 `/etc/renderd.conf`。用 nano 之類的文字編輯器開啟編輯：

```sh
sudo nano /etc/renderd.conf
```

在檔案最後加入類似以下的區段：

```ini
[s2o]
URI=/hot/
XML=/home/accountname/src/openstreetmap-carto/mapnik.xml
HOST=localhost
TILESIZE=256
MAXZOOM=20
```

XML 檔案的位置「/home/accountname/src/openstreetmap-carto/mapnik.xml」要改成你系統上的實際路徑。如果想要，也可以修改「[s2o]」和「URI=/hot/」。如果想從同一台伺服器渲染多組圖磚也沒問題，只要再加一個類似「[s2o]」的區段，使用不同名稱並指向不同的地圖樣式即可。也可以改用預設「gis」以外的資料庫，不過這超出本文件的範圍。如果記憶體只有 2Gb 左右，也應該把「num_threads」降到 2。選用「URI=/hot/」，是為了讓這裡產生的圖磚更容易用來取代 OpenStreetMap.org 上的 HOT 圖磚圖層。你也可以改用其他名稱，不過用「/hot/」也很好。

本教學最初撰寫時，Ubuntu 24.04 提供的 Mapnik 版本為 3.1，檔案中「[mapnik]」區段的「plugins_dir」設定為「/usr/lib/mapnik/3.1/input」。這個「3.1」日後可能還會改變。如果渲染圖磚時出現類似以下的錯誤：

!!! warning ""
    An error occurred while loading the map layer 's2o': Could not create datasource for type: 'postgis' (no datasource plugin directories have been successfully registered) encountered during parsing of layer 'landcover-low-zoom'

請查看「/usr/lib/mapnik」裡有哪些版本資料夾，並確認「/usr/lib/mapnik/(version)/input」中有「postgis.input」檔案。

設定好「renderd」如何回應圖磚渲染請求後，接下來要讓 Apache 網頁伺服器把請求轉送過去。可惜近期版本的 mod_tile 已移除這部分設定，不過目前仍可從這裡安裝：

```sh
cd /etc/apache2/conf-available/
sudo wget https://raw.githubusercontent.com/openstreetmap/mod_tile/python-implementation/etc/apache2/renderd.conf
sudo a2enconf renderd
sudo systemctl reload apache2
```

#### 確認可以看到除錯訊息

這時能看到圖磚渲染程序的輸出（包括任何錯誤）會很有幫助。近期版本的 mod_tile 預設關閉這項功能，開啟方式如下：

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

查看 /var/log/syslog，你應該會看到來自「renderd」服務的訊息。一開始會有一些字型警告，目前先不用理會。接著：

```sh
sudo /etc/init.d/apache2 restart
```

在 `syslog` 中應該會看到類似以下的訊息：

```log
Apr 23 11:14:10 servername apachectl[2031]: [Sat Apr 23 11:14:10.190678 2024] [tile:notice] [pid 2031:tid 140608477239168] Loading tile config s2o at /hot/ for zooms 0 - 20 from tile directory /var/cache/renderd/tiles with extension .png and mime type image/png
```

接下來，用網頁瀏覽器開啟「http://yourserveripaddress/index.html」（把 yourserveripaddress 換成伺服器的實際位址）。你應該會看到「Apache2 Ubuntu Default Page」。

如果不知道伺服器分配到哪個 IP 位址，通常可以用「ifconfig」查詢；網路設定不太複雜時，就是那個不是「127.0.0.1」的「inet addr」。如果你用的是主機服務商的伺服器，內部位址很可能和分配給你的外部位址不同，但服務商應該已經把外部 IP 位址寄給你，而且很可能就是你目前連線到伺服器的位址。

注意這只是「http」（連接埠 80）網站；要啟用 https 還需要額外的 Apache 設定，這超出本教學的範圍。不過如果用「Let's Encrypt」核發憑證，設定過程也能順便把 Apache 的 HTTPS 網站設定好。

接下來，用網頁瀏覽器開啟：http://yourserveripaddress/hot/0/0/0.png

如果你在上面修改了「URI=/hot/」，這裡當然也要跟著改。你應該會看到一張小小的世界地圖。如果沒有，請檢查顯示的錯誤訊息。常見原因是權限問題，或不小心漏掉了上面某些步驟。如果還是拿不到圖磚並出現其他錯誤，請把完整輸出貼到 pastebin，然後到 help.openstreetmap.org 之類的地方提問。

## 檢視圖磚

要看到圖磚，這裡偷個懶，用 HTML 檔案 `sample_leaflet.html` 顯示一張非常簡單的地圖。取得方式如下：

```sh
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo nano sample_leaflet.html
```

編輯檔案，把 IP 位址從「127.0.0.1」改成 yourserveraddress，讓其他電腦也能存取這台伺服器。然後瀏覽「http://yourserveraddress/sample_leaflet.html」。

第一次顯示地圖需要一點時間。你可以放大縮小，但依伺服器速度不同，有些圖磚一開始可能會是灰色，因為來不及渲染並送到瀏覽器。不過渲染完成後，下次就能直接使用。查看 /var/log/syslog 應該可以看到圖磚請求。

若要即時查看這些請求，可以在 ssh 連線中執行：

```sh
tail -f /var/log/syslog | grep " TILE "
```

（注意「TILE」前後的空格）

每收到一次圖磚請求會顯示一行，每渲染完一張圖磚也會顯示一行。

如果需要，可以把「/etc/apache2/conf-available/renderd.conf」中的「ModTileMissingRequestTimeout」設定從 10 秒提高到 30 或 60 秒，讓系統在背景渲染圖磚時等候更久，再回傳灰色圖磚給使用者。修改後記得執行「sudo service renderd restart」和「sudo service apache2 restart」。

恭喜！前往[使用圖磚](/using-tiles/index.md)章節，用新架好的圖磚伺服器做出一張地圖。
