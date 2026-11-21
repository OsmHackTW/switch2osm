---
layout: docs
title: 隨開放街圖編輯同步更新你的資料庫
lang: zh-TW
---

# 使用 PyOsmium 隨開放街圖編輯同步更新你的資料庫

每天都有數以百萬計的地圖更新；為了不讓地圖「過時」，可以定期更新用來產生地圖圖磚的資料。

用 osm2pgsql（1.4.2 或以上版本）來做，現在比以前容易得多。Ubuntu 22.04 與 Debian 12 以後的版本都內附合適的版本，也可以依照[這份說明](https://osm2pgsql.org/doc/install.html){: target=_blank}取得。

更簡單但較不具彈性的資料庫更新方法是使用「osm2pgsql-replication」，說明請見[這裡](/serving-tiles/updating-as-people-edit-osm2pgsql-replication.md)。這個範例會使用「PyOsmium」，以 planet.openstreetmap.org 的每分鐘更新，來更新最初從 Geofabrik 匯入的資料庫。

## 確認可以看到除錯訊息

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

## 初始化同步更新

重要提醒：下方使用的圖磚過期腳本假設圖磚寫在 `/var/cache/renderd/` 底下。如果 `/etc/renderd.conf` 指定了其他位置，就要先修改這裡建立的腳本，才能用它讓圖磚過期。由於這個資料夾一定存在，也會拿來存放「pyosmium」所需的工作檔案。

這個範例假設你是用以下方式匯入資料庫（`pgsql`）：

```sh
sudo -u _renderd \
    osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 3000 --number-processes 4 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/greater-london-latest.osm.pbf
```

或是這樣（`flex`）：

```sh
sudo -u _renderd \
    osm2pgsql -O flex \
    -S        ~/src/openstreetmap-carto/openstreetmap-carto-flex.lua \
    -d gis --create --slim  \
    -C 3000 --number-processes 4 \
    ~/data/greater-london-latest.osm.pbf
```

匯入的資料取自像[這樣的頁面](http://download.geofabrik.de/europe/united-kingdom/england/greater-london.html){: target=_blank}，頁面上會寫著類似「... and contains all OSM data up to 2026-06-12T20:21:15Z」的內容。下面設定同步更新時會用到這個日期：

```sh
sudo mkdir /var/cache/renderd/pyosmium
sudo chown _renderd /var/cache/renderd/pyosmium
sudo mkdir /var/log/tiles
sudo chown _renderd /var/log/tiles
cd /var/cache/renderd/pyosmium
sudo apt install pyosmium
sudo -u _renderd pyosmium-get-changes -D 2026-06-12T20:21:15Z -f sequence.state -v
```

最後一行會建立「sequence.state」檔案，這一行使用的日期必須與你下載的資料相符。

## 使用「trim_osc.py」只套用特定區域的更新

（選用，但如果更新的資料庫只包含地球上的某個區域，建議使用）

這裡使用 Zverik「regional」腳本中的「trim_osc.py」，把 OpenStreetMap.org 的更新裁剪到只剩感興趣的區域，以免 postgres 資料庫在套用更新時大幅膨脹。另外也要安裝一些相依套件。以你使用的非 root 使用者執行：

```sh
cd ~/src
git clone https://github.com/zverik/regional
chmod u+x ~/src/regional/trim_osc.py

sudo apt install python3-shapely python3-lxml
```

## 套用更新

實際套用更新的腳本已放在[這裡](https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/call_pyosmium.sh){: target=_blank}（`pgsql`）或[這裡](https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/call_pyosmium_flex.sh){: target=_blank}（`flex`）（如先前所述，兩者的 `osm2pgsql` 參數不同）。建議放在 `/usr/local/sbin`。腳本需要依環境做些調整：

* 如果你不使用「trim_osc.py」，只要移除「Trim the downloaded changes」註解與「The osm2pgsql append line」註解之間的程式碼即可。

* 如果你使用「trim_osc.py」，需要確認 TRIM_BIN 指向正確的位置，且 TRIM_REGION_OPTIONS 符合你感興趣的區域（腳本中的預設值涵蓋 IE+GB，即愛爾蘭與大不列顛）。

* 「osm2pgsql --append」的參數需要依你的資料庫（`pgsql` 或 `flex`）與所用的伺服器（分配的記憶體量、執行緒數量等）自訂。

* 傳給「render_expired」的參數需要自訂（要處理多少縮放等級，以及在各等級如何處理髒圖磚）。

顯示目前資料庫同步延遲的腳本可以在[這裡](https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/pyosmium_replag.sh){: target=_blank}取得，以 mod_tile 原始碼附帶的「mod_tile」範例為基礎。`pyosmium_replag.sh` 會以秒為單位顯示同步延遲；`pyosmium_replag.sh -h` 則以小時顯示（不到一小時則以分鐘或秒顯示）。建議也放在 `/usr/local/sbin`。別忘了讓兩個腳本都可執行。

執行腳本一次：

```sh
sudo -u _renderd /usr/local/sbin/call_pyosmium.sh
```

或

```sh
sudo -u _renderd /usr/local/sbin/call_pyosmium_flex.sh
```

執行時會在 `/var/cache/renderd/pyosmium` 中建立工作檔案（以及所執行指令的詳細輸出）。完成時應該會看到類似以下內容：

```log
Pyosmium update started:  Mon Jul 3 11:15:36 PM UTC 2023
Filtering newchange.osc.gz
Importing newchange.osc.gz
2023-07-03 23:17:44  osm2pgsql took 45s overall.
Expiring tiles

Total for all tiles rendered
Meta tiles rendered: Rendered 0 tiles in 1.54 seconds (0.00 tiles/s)
Total tiles rendered: Rendered 0 tiles in 1.54 seconds (0.00 tiles/s)
Total tiles in input: 2334224
Total tiles expanded from input: 305839
Total meta tiles deleted: 0
Total meta tiles touched: 0
Total tiles ignored (not on disk): 305839
Database Replication Lag: 1 day(s) and 1 hour(s)
```

要調整每次執行的下載量，可以修改腳本中 `pyosmium-get-changes` 的大小參數（`-s`）。單位是 MB，腳本預設為 20；若未指定，每次會下載 100MB。

執行更新的腳本可以加入 `_renderd` 帳號的 crontab：

```sh
sudo -u _renderd crontab -e
```

然後視情況加入：

```sh
*/5 *  *   *   *     /usr/local/sbin/call_pyosmium.sh >> /var/log/tiles/run.log
```

或

```sh
*/5 *  *   *   *     /usr/local/sbin/call_pyosmium_flex.sh >> /var/log/tiles/run.log
```

腳本在套用更新前會檢查自己是否已在執行，因此可以相當頻繁地執行；這個例子是每 5 分鐘一次。

建議在 renderd 重新啟動時清除「pyosmium 執行中」旗標。作法如下：

```sh
sudo nano /usr/lib/systemd/system/renderd.service
```

並在「[Service]」區段中加入：

```ini
ExecStartPre=rm -f /var/cache/renderd/pyosmium/call_pyosmium.running
```

然後：

```sh
sudo systemctl daemon-reload
sudo systemctl restart renderd
```

## 設定 munin

如果你使用 munin 回報「mod_tile」與「renderd」的活動，也可以讓它呼叫「pyosmium_replag.sh」來顯示資料庫同步延遲：

```sh
sudo nano /etc/munin/plugins/replication_delay
```

腳本內容可以從[這裡](https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/munin/replication_delay_pyosmium){: target=_blank}取得。它會使用我們先前建立的 `pyosmium_replag.sh` 取得以秒為單位的同步延遲。把這個腳本設成所有人都可執行，然後：

```sh
sudo /etc/init.d/munin-node restart
```

完成後不久，`http://yourserveraddress/munin/renderd-day.html` 應該會顯示「Data import lag」圖表。如果沒有，請查看 `/var/log/munin` 中的記錄。如需進一步協助釐清問題，請參考[這裡](https://guide.munin-monitoring.org/en/latest/develop/plugins/howto-write-plugins.html){: target=_blank}。
