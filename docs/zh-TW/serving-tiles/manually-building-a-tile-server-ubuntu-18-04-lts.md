---
layout: docs
title: 手動架設圖磚伺服器（18.04 LTS）
dist: Ubuntu 18.04
dl_timestamp: "2017-02-26T21:43:02Z"
lang: zh-TW
---

# {{ title }}

!!! info ""
    本頁說明如何安裝與設定執行自有圖磚伺服器所需的全部軟體。這份逐步教學以 [Ubuntu Linux](https://ubuntu.com/){: target=_blank} 18.04 LTS（Bionic Beaver）撰寫，並於 2020 年 5 月測試。

## 軟體安裝

OSM 圖磚伺服器堆疊是一組相互搭配、共同組成圖磚伺服器的程式與函式庫。和開放街圖（OpenStreetMap）的許多事情一樣，要達成這個目標有很多方法，而且幾乎每個元件都有各具優缺點的替代方案。本教學介紹的是最標準的版本，與 OpenStreetMap.org 主要圖磚伺服器所使用的架構相近。

這套堆疊由 5 個主要元件組成：`mod_tile`、`renderd`、`mapnik`、`osm2pgsql` 以及 `postgresql/postgis` 資料庫。Mod_tile 是 Apache 模組，負責提供已快取的圖磚，並判斷哪些圖磚需要重新渲染，原因可能是尚未快取，或是已經過時。Renderd 為不同類型的請求提供優先佇列系統，用來管理並分散渲染請求帶來的負載。Mapnik 則是實際負責渲染的軟體函式庫，供 renderd 呼叫。

注意本教學是在全新安裝的 {{ dist }} 伺服器上撰寫並測試。如果你已經安裝了某些軟體的其他版本（例如從舊版 Ubuntu 升級，或另外設定了 PPA 來源），可能需要自行做些調整。

本教學假設你以非 root 使用者搭配 `sudo` 執行所有操作。下文預設使用的非 root 使用者名稱是 `renderaccount`，你可以在本機建立這個帳號，也可以修改腳本改用其他使用者名稱。如果建立了 `renderaccount` 使用者，要把它加入可以 `sudo` 成 root 的群組。在你平常使用的非 root 帳號下執行：

```sh
sudo -i
usermod -aG sudo renderaccount
exit
```

要建置這些元件，必須先安裝各種相依套件：

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-deps.txt"
```

詢問時回答 yes 即可安裝。這會花上一段時間，先去泡杯茶吧。這份清單包含各種工具程式與函式庫、Apache 網頁伺服器，以及「carto」，用來把 Carto-CSS 樣式表轉換成地圖渲染器「mapnik」看得懂的格式。完成後，再安裝第二組必要套件：

## 安裝 postgresql / postgis

Ubuntu 有預先打包好的 `postgis` 與 `postgresql`，可以直接用 Ubuntu 套件管理員安裝。

```sh
sudo apt-get install \
    postgresql \
    postgresql-contrib \
    postgis \
    postgresql-10-postgis-2.4 \
    postgresql-10-postgis-scripts
```

這裡 `postgresql` 是用來儲存地圖資料的資料庫，`postgis` 則為它加上額外的空間資料支援。同樣回答 yes 即可安裝。

接著要建立 postgis 資料庫。許多程式的預設值都假設資料庫名稱是 `gis`，本教學也沿用這個慣例，不過這不是必要的。下文中的 `renderaccount` 請換成你的使用者名稱，也就是之後要用 Mapnik 渲染地圖的使用者。

```sh
sudo -u postgres -i
createuser renderaccount
createdb -E UTF8 -O renderaccount gis
```

仍以 `postgres` 使用者身分，在 PostgreSQL 資料庫上設定 PostGIS（同樣把下方的 `renderaccount` 換成你的使用者名稱）：

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
ALTER TABLE geometry_columns OWNER TO renderaccount;
```

（會回應 ALTER TABLE）

```sql
ALTER TABLE spatial_ref_sys OWNER TO renderaccount;
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

如果還沒有這個使用者的 Unix 帳號，也請建立一個，並在提示時設定密碼：

```sh
sudo useradd -m renderaccount
sudo passwd renderaccount
```

同樣地，請把上面的 `renderaccount` 換成你選用的非 root 使用者名稱。

### 安裝 osm2pgsql

有好幾個軟體需要從原始碼安裝，第一個是 `osm2pgsql`。把開放街圖資料匯入資料庫並加以管理的工具有很多，這裡使用的 `osm2pgsql` 大概是最熱門的一個。

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/openstreetmap/osm2pgsql
cd osm2pgsql
```

osm2pgsql 的建置方式和舊版不同，因此還要再安裝一些必要套件：

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-osm2pgsql-deps.txt"
```

同樣回答 yes 即可安裝。

```sh
mkdir build && cd build
cmake ..
```

（輸出結尾應該是「build files have been written to」）

```sh
make
```

（輸出結尾應該是「[100%] Built target osm2pgsql」）

```sh
sudo make install
```

## Mapnik

接下來安裝 Mapnik，這裡使用 {{ dist }} 的預設版本：

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-mapnik-deps.txt"
```

檢查 Mapnik 是否正確安裝：

```py
python
>>> import mapnik
>>>
```

如果 Python 回應第二個 `>>>` 提示字元且沒有錯誤，就表示 Python 找得到 Mapnik 函式庫。恭喜！用以下指令即可離開 Python：

```py
>>> quit()
```

## 安裝 mod_tile 與 renderd

接下來安裝 mod_tile 與 renderd。`mod_tile` 是處理圖磚請求的 Apache 模組；`renderd` 則是常駐程式，在 `mod_tile` 提出請求時實際渲染圖磚。這裡使用 <https://github.com/SomeoneElseOSM/mod_tile>{: target=_blank} 的 `switch2osm` 分支。它分叉自 <https://github.com/openstreetmap/mod_tile>{: target=_blank}，修改後支援 {{ dist }}，另外還做了幾項變更，讓它能在一般 Ubuntu 伺服器上運作，而不限於 OSM 的渲染伺服器。

### 編譯 mod_tile 原始碼

```sh
cd ~/src
git clone -b switch2osm git://github.com/SomeoneElseOSM/mod_tile.git
cd mod_tile
./autogen.sh
```

（結尾應該是 `autoreconf: Leaving directory '.'`。）

```sh
./configure
```

（結尾應該是 `config.status: executing libtool commands`）

```sh
make
```

注意這時畫面上會捲過一些「看起來令人擔心」的訊息，但結尾應該是「make[1]: Leaving directory '/home/renderaccount/src/mod_tile'」。

```sh
sudo make install
```

（結尾應該是「make[1]: Leaving directory '/home/renderaccount/src/mod_tile'」）

```sh
sudo make install-mod_tile
```

（結尾應該是「chmod 644 /usr/lib/apache2/modules/mod_tile.so」）

```sh
sudo ldconfig
```

（應該不會有任何回應）

## 樣式表設定

所有必要軟體都安裝好之後，接著要下載並設定樣式表。

這裡使用的樣式就是 openstreetmap.org 網站「標準」地圖所用的樣式。選擇它是因為文件完整，而且在世界各地都能正常運作（包括地名使用非拉丁文字的地區）。不過它也有些缺點：為了全球通用，設計上做了許多妥協；如果需要修改，結構也相當複雜，不容易理解。

「OpenStreetMap Carto」的官方網址是 <https://github.com/gravitystorm/openstreetmap-carto/>{: target=_blank}，安裝說明則在 <https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md>{: target=_blank}，不過本教學會涵蓋所有必要步驟。

這裡假設樣式表放在 `renderaccount` 使用者（或你使用的其他帳號）的 `~/src` 底下某個資料夾：

```sh
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
```

接下來安裝合適版本的 `carto` 編譯器。這個版本比 Ubuntu 內建的還新，因此需要執行：

```sh
sudo apt-get install nodejs-dev node-gyp libssl1.0-dev
```

有點令人困惑的是，這會移除一些先前安裝的套件。接著執行：

```sh
sudo apt install npm nodejs
sudo npm install -g carto
carto -v
```

顯示的版本號應不低於：

```sh
carto 1.2.0 (Carto map stylesheet compiler)
```

然後把 carto 專案轉換成 Mapnik 看得懂的格式：

```sh
carto project.mml > mapnik.xml
```

現在 `/home/renderaccount/src/openstreetmap-carto/mapnik.xml` 已經有一份 Mapnik XML 樣式表。

## 匯入資料

一開始先只匯入少量測試資料。可以下載的地方很多，其中 `download.geofabrik.de` 的選擇相當豐富。這個範例會下載亞塞拜然的資料，目前大約 32Mb。

瀏覽 <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank}，記下「This file was last modified」的日期（例如「{{ dl_timestamp }}」）。之後如果要把其他人對開放街圖的後續修改更新進資料庫，就會用到這個日期。下載方式如下：

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

以下指令會把剛才下載的開放街圖資料匯入資料庫。這個步驟對硬碟 I/O 的負擔很重；依硬體而定，匯入整個地球可能要花好幾個小時、好幾天甚至好幾週。較小的節選資料匯入起來會快很多，你也可能需要試試不同的 `-C` 值，配合機器可用的記憶體。

```sh
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
: 決定多重多邊形的處理方式。

`--hstore`
: 讓沒有對應資料庫欄位的標籤也能用於渲染。

`--tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua`
: 指定用來處理標籤的 Lua 腳本。這是在樣式處理 OSM 標籤之前先行處理標籤的簡便方法，有機會讓樣式邏輯簡化許多。

`-C 2500`
: 為 osm2pgsql 匯入程序分配 2.5 Gb 記憶體。如果記憶體較少，可以試試較小的數值；如果匯入程序因記憶體不足而被終止，就需要改用更小的數值或更小的 OSM 節選資料。

`--number-processes 1`
: 使用 1 顆 CPU。如果有更多核心可用，可以多用一些。

`-S ~/src/openstreetmap-carto/openstreetmap-carto.style`
: 依這個檔案建立資料庫欄位（內容其實與「openstreetmap-carto」原本的相同，沒有更動）

`~/data/azerbaijan-latest.osm.pbf`
: 最後一個參數是要匯入的資料檔。

指令完成時會顯示類似「Osm2pgsql took 238s overall」的訊息。

### 下載 Shapefile

雖然繪製地圖用的資料大多直接來自上面下載的開放街圖資料檔，但仍需要一些 shapefile，例如低縮放等級的國界。下載並建立索引：

```sh
cd ~/src/openstreetmap-carto/
scripts/get-external-data.py
```

這個過程要下載相當多的資料，可能需要一些時間，執行時畫面上不會顯示太多訊息。實際上它會在「openstreetmap-carto」底下建立「data」資料夾並放入資料。

### 字型

Carto v5.6.0 以上的版本需要手動安裝字型：

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

選擇亞塞拜然當測試資料區域，一方面是因為範圍小，另一方面是當地有些地名含有非拉丁字元。

## 設定網頁伺服器

### 設定 renderd

`renderd` 的設定檔是 `/usr/local/etc/renderd.conf`。用 nano 之類的文字編輯器開啟編輯：

```sh
sudo nano /usr/local/etc/renderd.conf
```

其中有幾行可能需要修改。在 `renderd` 區段：

```ini
num_threads=4
```

如果記憶體只有 2Gb 左右，建議降到 2。`ajt` 區段對應名為 `ajt` 的「具名地圖樣式」。只要每個區段的 URI 不同，就可以有多個這樣的區段。`XML` 這一行需要改成類似：

```ini
XML=/home/renderaccount/src/openstreetmap-carto/mapnik.xml
```

請把 `renderaccount` 改成你上面使用的非 root 使用者名稱。

```ini
URI=/hot/
```

選用這個值，是為了讓這裡產生的圖磚更容易用來取代 OpenStreetMap.org 上的 HOT 圖磚圖層。你也可以改用其他值，不過用 `/hot/` 也很好。

### 設定 Apache

```sh
sudo mkdir /var/lib/mod_tile
sudo chown renderaccount /var/lib/mod_tile
```

```sh
sudo mkdir /var/run/renderd
sudo chown renderaccount /var/run/renderd
```

現在要讓 Apache 認得 `mod_tile`，用 nano（或其他編輯器）開啟：

```sh
sudo nano /etc/apache2/conf-available/mod_tile.conf
```

在檔案中加入下列這行：

```ini
LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so
```

存檔後執行：

```sh
sudo a2enconf mod_tile
```

畫面會提示需要執行 `service apache2 reload` 才能啟用新設定，先不用急著執行。

接著要讓 Apache 認得 `renderd`，用 nano（或其他編輯器）開啟：

```sh
sudo nano /etc/apache2/sites-available/000-default.conf
```

在 `ServerAdmin` 與 `DocumentRoot` 兩行之間加入下列內容：

```ini
LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# 等待圖磚渲染完成的逾時時間，超過即放棄
ModTileRequestTimeout 0
# 等待原本不存在的圖磚渲染完成的逾時時間，超過即放棄
ModTileMissingRequestTimeout 30
```

然後重新載入 Apache 兩次：

```sh
sudo service apache2 reload
sudo service apache2 reload
```

（之所以要執行兩次，我猜是因為 Apache 在執行中重新設定時會「搞混」）

用網頁瀏覽器開啟 `http://your.server.ip.address/index.html`，應該會看到 Ubuntu / Apache 的「It works!」頁面。

!!! tip
    如果不知道伺服器分配到哪個 IP 位址，通常可以用 `ifconfig` 查詢；網路設定不太複雜時，就是 `127.0.0.1` 以外的那個 `inet addr`。

如果你用的是主機服務商的伺服器，內部位址很可能和分配給你的外部位址不同，但服務商應該已經把外部 IP 位址寄給你，而且很可能就是你目前連線到伺服器的位址。

注意這只是 `http`（連接埠 80）網站；要啟用 `https` 還需要額外的 Apache 設定，這超出本說明的範圍。不過如果用「Let's Encrypt」核發憑證，設定過程也能順便把 Apache 的 HTTPS 網站設定好。

### 首次執行 renderd

接下來執行 `renderd`，試著渲染一些圖磚。一開始先在前景執行，方便即時看到錯誤：

```sh
renderd -f -c /usr/local/etc/renderd.conf
```

這時可能會看到一些警告，暫時不用理會，但不應該出現任何錯誤。如果有錯誤，請把完整輸出貼到 Pastebin，然後到 [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank} 之類的地方提問（附上 Pastebin 連結，不要把全部文字貼在問題裡）。

用網頁瀏覽器開啟：`http://yourserveripaddress/hot/0/0/0.png`

瀏覽器中應該會出現一張世界地圖，指令列上也會出現更多除錯訊息，包括「DEBUG: START TILE」和「DEBUG: DONE TILE」。「DEBUG: Failed to read cmd on fd」訊息可以忽略，那不是錯誤。如果沒有出現圖磚，又出現其他錯誤，請把完整輸出貼到 Pastebin，然後到 [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank} 之類的地方提問。

如果一切正常，按 ++control+c++ 停止前景渲染程序。

### 在背景執行 renderd

接下來設定 `renderd` 在背景執行。首先編輯 `~/src/mod_tile/debian/renderd.init` 檔案，把「RUNASUSER」設為先前使用的非 root 帳號，例如 `renderaccount`，然後複製到系統資料夾：

```sh
nano ~/src/mod_tile/debian/renderd.init
sudo cp ~/src/mod_tile/debian/renderd.init /etc/init.d/renderd
sudo chmod u+x /etc/init.d/renderd
sudo cp ~/src/mod_tile/debian/renderd.service /lib/systemd/system/
```

`renderd.service` 是 `systemd` 服務檔，這裡的版本只是去呼叫舊式的 init 指令。測試啟動指令是否正常：

```sh
sudo /etc/init.d/renderd start
```

（應該會回應「[ ok ] Starting renderd (via systemctl): renderd.service」。）

設定成每次開機自動啟動：

```sh
sudo systemctl enable renderd
```

## 檢視圖磚

要看到圖磚，這裡偷個懶，直接用 mod_tile「extras」資料夾中的 HTML 檔案 `sample_leaflet.html`。只要在架設圖磚伺服器的機器上用網頁瀏覽器開啟這個檔案即可。如果伺服器上沒有網頁瀏覽器可用，可以編輯這個檔案，把 `127.0.0.1` 換成伺服器的 IP 位址，再複製到 `/var/www/html` 底下。

在 ssh 連線中執行：

```sh
tail -f /var/log/syslog | grep " TILE "
```

（注意 `" TILE "` 前後的空格）

每收到一次圖磚請求會顯示一行，每渲染完一張圖磚也會顯示一行。

載入頁面時應該會看到一些圖磚請求。逐步縮小地圖時，ssh 連線中會出現新圖磚的請求。有些低縮放等級的圖磚第一次渲染可能要花很久（好幾分鐘），但完成後下次就能直接使用。

恭喜！前往[使用圖磚](/using-tiles/index.md)章節，用新架好的圖磚伺服器做出一張地圖。
