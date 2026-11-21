---
layout: docs
title: 隨開放街圖編輯同步更新你的資料庫
lang: zh-TW
---

# 使用 osm2pgsql 隨開放街圖編輯同步更新你的資料庫

每天都有數以百萬計的地圖更新；為了不讓地圖「過時」，可以定期更新用來產生地圖圖磚的資料。

用 osm2pgsql（1.4.2 或以上版本）來做，現在比以前容易得多。Ubuntu 22.04 內附 1.6.0 版，Debian 12 內附 1.8.0 版，也可以依照[這份說明](https://osm2pgsql.org/doc/install.html){: target=_blank}取得。osm2pgsql 附帶 [osm2pgsql-replication](https://osm2pgsql.org/doc/manual.html#updating-an-existing-database){: target=_blank}，提供相對簡單的方式讓資料庫保持最新。更有彈性的做法是直接呼叫 PyOsmium，作法請參考[這份教學](/serving-tiles/updating-as-people-edit-pyosmium.md)。

你可以從許多不同的來源設定同步更新。開放街圖（OpenStreetMap）本身提供每分鐘、每小時與每日的更新，其他來源如 Geofabrik 則可提供每日更新，範圍對應 [download.geofabrik.de](http://download.geofabrik.de/index.html){: target=_blank} 上提供的區域節選。

## 使用 Geofabrik 資料的實作範例

這個範例會從 Geofabrik 下載倫敦的資料，匯入資料庫，再設定更新。Geofabrik 的更新只涵蓋你下載的區域，每天提供一次。

首先，以你使用的非 root 帳號下載資料：

```sh
# 若資料夾尚不存在
mkdir ~/data
cd ~/data
wget http://download.geofabrik.de/europe/united-kingdom/england/greater-london-latest.osm.pbf
```

接著匯入資料庫。實際參數取決於使用 `osm2pgsql` 的 `pgsql` 輸出（OSM Carto 5.9.0 以前的版本使用）還是 `flex` 輸出（OSM Carto 6.0.0 以上版本使用）。另外，這裡的處理程序數與記憶體數值可依系統調整。

`pgsql`：

```sh
sudo -u _renderd \
    osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 3000 --number-processes 4 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/greater-london-latest.osm.pbf
```

`flex`：

```sh
sudo -u _renderd \
    osm2pgsql -O flex \
    -S        ~/src/openstreetmap-carto/openstreetmap-carto-flex.lua \
    -d gis --create --slim  \
    -C 3000 --number-processes 4 \
    ~/data/greater-london-latest.osm.pbf
```

重要提醒：下方使用的圖磚過期腳本假設圖磚寫在 `/var/cache/renderd/` 底下。如果 `/etc/renderd.conf` 指定了其他位置，就要先修改這裡建立的腳本，才能用它讓圖磚過期。

接著初始化同步更新。如[這裡](https://osm2pgsql.org/doc/manual.html#updating-an-existing-database){: target=_blank}所述，從 Geofabrik（以及 [download.openstreetmap.fr](https://download.openstreetmap.fr/){: target=_blank}）下載的檔案包含同步更新所需的所有資訊（下載更新的網址與起始日期）。

```sh
sudo -u _renderd \
    osm2pgsql-replication init \
    -d gis \
    --osm-file ~/data/greater-london-latest.osm.pbf
```

輸出大致如下：

```log
2022-04-24 23:32:42 [INFO]: Initialised updates for service 'http://download.geofabrik.de/europe/united-kingdom/england/greater-london-updates'.
2022-04-24 23:32:42 [INFO]: Starting at sequence 3314 (2022-04-23 20:21:53+00:00).
```

在這個範例中，顯示的日期就是下載資料時[這個頁面](http://download.geofabrik.de/europe/united-kingdom/england/greater-london.html){: target=_blank}上的日期。

### 建立套用更新的腳本

接下來要建立寫入過期記錄檔的位置，以及用來套用待處理更新、讓受影響圖磚過期的腳本。圖磚過期時，可以選擇重新渲染、標記為髒圖磚（dirty），或直接刪除。

```sh
sudo mkdir /var/log/tiles
sudo chown _renderd /var/log/tiles
sudo nano /usr/local/sbin/expire_tiles.sh
```

這個腳本的範例大致如下：

```sh title="expire_tiles.sh"
#!/bin/bash
render_expired --map=s2o --min-zoom=13 --max-zoom=20 \
    -s /run/renderd/renderd.sock < /var/cache/renderd/dirty_tiles.txt
rm /var/cache/renderd/dirty_tiles.txt
```

「髒」圖磚清單必須讓執行這個腳本的「_renderd」帳號能寫入。其他參數的可用設定請參考[man 說明頁面](https://manpages.ubuntu.com/manpages/jammy/en/man1/render_expired.1.html){: target=_blank}。上面的範例會嘗試重新渲染縮放等級 13 以上的所有髒圖磚。比較貼近實務的範例大致如下：

```sh title="expire_tiles.sh"
#!/bin/bash
render_expired --map=s2o --min-zoom=13 --touch-from=13 --delete-from=19 --max-zoom=20 \
    -s /run/renderd/renderd.sock < /var/cache/renderd/dirty_tiles.txt
rm /var/cache/renderd/dirty_tiles.txt
```

這與舊版 osmosis 腳本使用的[預設值](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/openstreetmap-tiles-update-expire#L58){: target=_blank}相同：縮放等級 12 以下的圖磚不處理，13 到 19 標記為髒圖磚，20 則直接刪除。這個範例也可以在[這裡](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/expire_tiles.sh){: target=_blank}找到。

接著：

```sh
sudo nano /usr/local/sbin/update_tiles.sh
```

一開始，這個腳本的內容大致如下（`pgsql`）：

```sh title="update_tiles.sh"
#!/bin/bash
osm2pgsql-replication \
    update -d gis \
    --post-processing /usr/local/sbin/expire_tiles.sh \
    --max-diff-size 10  --  \
    -G --hstore \
    --tag-transform-script \
        /home/renderaccount/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 3000 --number-processes 4 \
    -S /home/renderaccount/src/openstreetmap-carto/openstreetmap-carto.style \
    --expire-tiles=1-20 \
    --expire-output=/var/cache/renderd/dirty_tiles.txt
```

`flex` 版本大致如下：

```sh title="update_tiles.sh"
#!/bin/bash                                                                                                   
osm2pgsql-replication \
    update -d gis \
    --post-processing /usr/local/sbin/expire_tiles.sh \
    --max-diff-size 10  --  \
    -S /home/renderaccount/src/openstreetmap-carto/openstreetmap-carto-flex.lua \
    -C 2500 --number-processes 4 \
    --expire-tiles=1-20 \
    --expire-output=/var/cache/renderd/dirty_tiles.txt
```

請依你使用的帳號，把 `renderaccount` 改成實際位置。

`osm2pgsql-replication update` 這行中 `--` 之後的所有內容都會當作參數傳給 osm2pgsql，必須與當初匯入資料庫時使用的參數一致。在 `--` 之前，`-d gis` 只是指定使用的資料庫，`--max-diff-size 10` 指定一次處理多少資料；不過要注意，osm2pgsql-replication 其實會反覆下載資料並更新資料庫，直到沒有資料要處理為止，可能要花一些時間。`--max diff-size` 決定每一輪擷取多少資料。`--post-processing /usr/local/sbin/expire_tiles.sh` 則只是呼叫我們另一個腳本。

讓兩個腳本都可執行：

```sh
sudo chmod ugo+x /usr/local/sbin/update_tiles.sh
sudo chmod ugo+x /usr/local/sbin/expire_tiles.sh
```

### 確認可以看到除錯訊息

這時能看到圖磚渲染程序的輸出會很有幫助，可以確認標記為髒的圖磚確實有在處理。近期的 mod_tile 版本預設關閉這項功能，開啟方式如下：

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

### 測試

在另一個工作階段中：

```sh
sudo tail -f /var/log/syslog
```

如果系統預設沒有「syslog」檔案（例如 Debian 12 以上版本），改為執行：

```sh
sudo journalctl -ef
```

現在應該會看到一些對應 mod_tile 請求的輸出。

執行腳本一次：

```sh
sudo -u _renderd /usr/local/sbin/update_tiles.sh
```

如果沒有待處理的更新（這些檔案的 Geofabrik 更新每天發布一次），會顯示類似以下內容：

```log
2023-07-02 16:57:03 [INFO]: Using replication service 'http://download.geofabrik.de/europe/britain-and-ireland-updates'. Current sequence 3743 (2023-07-01 20:21:30+00:00).
2023-07-02 16:57:03 [INFO]: Database already up-to-date.
```

如果有待處理的更新，則會看到類似以下內容：

```log
2022-06-05 16:29:32 [INFO]: Using replication service 'http://download.geofabrik.de/europe/united-kingdom/england/greater-london-updates'. Current sequence 3355 (2022-06-03 20:21:26+00:00).
2022-06-05 16:29:33  osm2pgsql version 1.6.0
2022-06-05 16:29:33  Database version: 14.3 (Ubuntu 14.3-0ubuntu0.22.04.1)
2022-06-05 16:29:33  PostGIS version: 3.2
2022-06-05 16:29:34  Setting up table 'planet_osm_point'
2022-06-05 16:29:34  Setting up table 'planet_osm_line'
2022-06-05 16:29:34  Setting up table 'planet_osm_polygon'
2022-06-05 16:29:34  Setting up table 'planet_osm_roads'
2022-06-05 16:29:51  Reading input files done in 17s.
2022-06-05 16:29:51    Processed 4267 nodes in 7s - 610/s
2022-06-05 16:29:51    Processed 1108 ways in 7s - 158/s
2022-06-05 16:29:51    Processed 4 relations in 3s - 1/s
2022-06-05 16:29:52  Going over 993 pending ways (using 4 threads)
Left to process: 0.....
2022-06-05 16:29:57  Processing 993 pending ways took 5s at a rate of 198.60/s
2022-06-05 16:29:57  Going over 129 pending relations (using 4 threads)
...
```

有圖磚需要過期時，會出現類似這樣的一行：

```log
Read and expanded 42700 tiles from list.
render: file:///var/cache/renderd/tiles/s2o/18/17/245/244/200/0.meta
Read and expanded 42800 tiles from list.
```

最後會印出總計：

```log
Read and expanded 121200 tiles from list.
Read and expanded 121300 tiles from list.
Waiting for rendering threads to finish

Total for all tiles rendered
Meta tiles rendered: Rendered 1 tiles in 18.19 seconds (0.05 tiles/s)
Total tiles rendered: Rendered 64 tiles in 18.19 seconds (3.52 tiles/s)
Total tiles in input: 121357
Total tiles expanded from input: 10766
Total meta tiles deleted: 0
Total meta tiles touched: 0
Total tiles ignored (not on disk): 10765
2022-06-05 16:36:58 [INFO]: Data imported until 2022-06-04 20:21:41+00:00. Backlog remaining: 20:15:17.919969
```

記錄檔中的輸出會包含類似以下內容：

```log
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Data is available now on 1 fds
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Got incoming connection, fd 5, number 0, total conns 1, total slots 1
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Data is available now on 1 fds
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Got incoming request with protocol version 2
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Got command RenderBulk fd(5) xml(s2o), z(18), x(131008), y(87168), mime(image/png), options()
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: START TILE s2o 18 131008-131015 87168-87175, age 0.04 days
Jun  5 16:36:40 ubuntuvm75 renderd[5838]: Rendering projected coordinates 18 131008 87168 -> -9783.939621|6710559.587216 -8560.947168|6711782.579668 to a 8 x 8 tile
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: DONE TILE s2o 18 131008-131015 87168-87175 in 18.046 seconds
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Creating and writing a metatile to /var/cache/renderd/tiles/s2o/18/17/245/244/200/0.meta
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Sending message Done to 5
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Sending render cmd(3 s2o 18/131008/87168) with protocol version 2 to fd 5
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Data is available now on 1 fds
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Failed to read cmd on fd 5
Jun  5 16:36:58 ubuntuvm75 renderd[5838]: Connection 0, fd 5 closed, now 0 left, total slots 1
```

### 每天執行

執行更新的腳本可以加入 root 的 crontab。首先修改 `update_tiles.sh`，讓它啟動時做些錯誤檢查，並只把摘要附加到記錄檔。你可以從[這裡](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/update_tiles.sh){: target=_blank}取得 `pgsql` 版本，或從[這裡](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/update_tiles_flex.sh){: target=_blank}取得 `flex` 版本（如上所述，兩者的 `osm2pgsql` 參數不同）。同樣請把「renderaccount」改成你在這裡使用的非 root 帳號名稱。你也可以調整執行緒數量與記憶體快取用量。

然後在非 root 帳號的 crontab 中加入類似以下的內容：

```sh
04 09  *   *   *     /usr/local/sbin/update_tiles.sh >> /var/log/tiles/run.log
```

這個範例會在每天早上 09:04 執行一次。由於 Geofabrik 的更新每天只發布一次，一天執行超過一次並沒有意義。

## 使用 openstreetmap.org 的每分鐘更新

openstreetmap.org 也提供更新。這些更新涵蓋全世界，因此會比同一時段單一區域（例如 Geofabrik 提供的）的更新更大。設定方式如下：

```sh
sudo -u _renderd \
    osm2pgsql-replication init -d gis  \
    --server https://planet.openstreetmap.org/replication/minute
```

會回傳類似以下內容：

```log
2022-06-05 20:49:21 [INFO]: Initialised updates for service 'https://planet.openstreetmap.org/replication/minute'.
2022-06-05 20:49:21 [INFO]: Starting at sequence 5088118 (2022-06-04 17:11:02+00:00).
```

更新的起始日期是由資料庫中最新的路徑（way）物件推得，通常比「此節選包含截至某日的所有 OSM 資料」的截止日期稍早一點。如果資料不是來自 OSM，就會出錯。另外也有每小時與每日的更新，所以也可以改用每小時或每日版本：

```sh
sudo -u _renderd \
    osm2pgsql-replication init -d gis  \
    --server https://planet.openstreetmap.org/replication/hour
```

```sh
sudo -u _renderd \
    osm2pgsql-replication init -d gis  \
    --server https://planet.openstreetmap.org/replication/day
```

初始化完成後，依上方「建立套用更新的腳本」設定同樣的腳本，並用同樣方式測試：

```sh
sudo -u _renderd /usr/local/sbin/update_tiles.sh
```

再次提醒，「osm2pgsql-replication」其實會反覆下載資料並更新資料庫，直到沒有資料要處理為止，而腳本中的 `--max diff-size` 決定每一輪擷取多少資料。最後完成時，結尾類似：

```log
2022-06-05 22:30:47 [INFO]: Data imported until 2022-06-05 21:45:42+00:00. Backlog remaining: 0:45:05.948787
```

執行更新的腳本可以照上面的方式修改，把摘要輸出到記錄檔，再加入非 root 帳號的 crontab：

```sh
*/5 *  *   *   *     /usr/local/sbin/update_tiles.sh >> /var/log/tiles/run.log
```

由於用的是每分鐘更新，而且腳本在套用更新前會檢查自己是否已在執行，因此可以比一天一次更頻繁地執行；這個例子是每 5 分鐘一次。

建議在 renderd 重新啟動時清除「osm2pgsql-replication 執行中」旗標。作法如下：

```sh
sudo nano /usr/lib/systemd/system/renderd.service
```

並在「[Service]」區段中加入：

```ini
ExecStartPre=rm -f /var/cache/renderd/update_tiles.sh.running
```

然後：

```sh
sudo systemctl daemon-reload
sudo systemctl restart renderd
```
