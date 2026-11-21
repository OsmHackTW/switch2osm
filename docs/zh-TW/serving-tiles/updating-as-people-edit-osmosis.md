---
layout: docs
title: 隨開放街圖編輯同步更新你的資料庫
lang: zh-TW
---

# 使用 Osmosis 隨開放街圖編輯同步更新你的資料庫

每天都有數以百萬計的地圖更新；為了不讓地圖「過時」，可以定期更新用來產生地圖圖磚的資料。

如果你使用的是近期版本的 osm2pgsql（1.4.2 或以上版本），或許可以改用[新的](/serving-tiles/updating-as-people-edit-osm2pgsql-replication.md)說明。如果想使用與下載區域完全吻合的更新來源（例如 Geofabrik 為自家下載區域提供的每日更新），那份說明特別有用。如果無法用 `osm2pgsql` 做同步更新，請繼續往下看……

這裡使用 `osmosis`，安裝時會一併裝好 Java。`osmosis` 是通用的 OSM 資料工具，其中一項功能就是用 OSM 的最新變更更新資料庫。其他選擇還有 `osmium` 等。

```sh
sudo apt install osmosis
```

（最後會顯示「done.」）

這裡使用 Zverik「regional」腳本中的 `trim_osc.py`，把 OpenStreetMap.org 的更新裁剪到只剩感興趣的區域，以免 postgres 資料庫在套用更新時大幅膨脹。另外也要安裝一些相依套件。

```sh
cd ~/src
git clone https://github.com/zverik/regional
chmod u+x ~/src/regional/trim_osc.py

sudo apt install python3-shapely python3-lxml
```

如果你是依照 Ubuntu 的說明操作，就已經有下一步所需的腳本。若是 Debian，則需要手動取得：

```sh
cd ~/src
git clone -b switch2osm https://github.com/SomeoneElseOSM/mod_tile.git
cd mod_tile
```

`openstreetmap-tiles-update-expire` 腳本用來從開放街圖取得更新。其中有這一行：

```sh
TRIM_REGION_OPTIONS="-b -14.17 48.85 2.12 61.27"
```

這大致對應英國的範圍，你可能會想改成其他數值。也可以用 `.poly` 檔案定義邊界。

其中還有：

```sh
ACCOUNT=renderaccount
```

這裡要改成存放 `mod_tile` 與「regional」原始碼的非 root 使用者名稱。在 Debian 11 上，這個帳號和用來渲染圖磚的帳號不同。

另外也請確認腳本中沒有 `-e$EXPIRY_METAZOOM:$EXPIRY_METAZOOM`。如果有，請把 `:` 改成 `-`：`-e$EXPIRY_METAZOOM-$EXPIRY_METAZOOM`。

另外，你可能需要編輯 OSM2PGSQL_OPTIONS，讓它符合匯入資料庫時所用的資料庫類型與 Lua 標籤轉換腳本；如果是 `pgsql`，還要符合決定建立哪些欄位的 `.style` 檔案。依建立資料庫時使用的參數不同，這裡可能還要再傳入其他參數。當然，請把 `/path/to/` 改成實際路徑：

（`pgsql`）

```sh
OSM2PGSQL_OPTIONS="-d $DBNAME --tag-transform-script /path/to/src/openstreetmap-carto/openstreetmap-carto.lua -S /path/to/src/openstreetmap-carto/openstreetmap-carto.style"
```

（`flex`）

```sh
OSM2PGSQL_OPTIONS="-d $DBNAME -S /path/to/openstreetmap-carto/openstreetmap-carto-flex.lua"
```

接著建立存放圖磚過期記錄的資料夾，並把擁有者改成用來渲染圖磚的使用者名稱（Debian 11 以上版本與較新的 Ubuntu 為 `_renderd`）。腳本也必須以這個帳號執行；由於這個帳號可能不是軟體下載位置的擁有者，這裡會傳入腳本的完整路徑，請把 `/path/to` 改成正確的值。

傳給 `openstreetmap-tiles-update-expire` 的參數應該是最初匯入資料的時間。如果你是從 Geofabrik 下載資料，就是下載時頁面（例如 <http://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank}）上「and contains all OSM data up」所顯示的日期。

```sh
sudo mkdir /var/log/tiles
sudo chown _renderd /var/log/tiles
sudo -u _renderd /path/to/src/mod_tile/openstreetmap-tiles-update-expire 2026-06-12T20:21:15Z
```

輸出的最後一行應該類似：

```log
2020-11-15 23:50:16 (2.62 MB/s) - ‘/var/lib/mod_tile/.osmosis/state.txt’ saved [342/342]
```

這會在 `/var/lib/mod_tile` 中建立 `.osmosis` 資料夾，記錄最後匯入資料的詳細資訊。

在一個連到伺服器的 ssh 工作階段中執行：

```sh
tail -f /var/log/tiles/run.log
```

在另一個工作階段中執行：

```sh
sudo -u _renderd /path/to/src/mod_tile/openstreetmap-tiles-update-expire
```

如先前一樣改成正確的帳號與路徑，但這次不帶參數。

如果你看到類似這樣的訊息：

```log
openstreetmap-tiles-update-expire: 1: python: not found
```

不用擔心，這只是磁碟空間檢查。

記錄檔的輸出應該類似：

```log
[2020-11-15 23:58:28] 3850 start import from seq-nr 4283484, replag is 1 day(s) and 2 hour(s)
[2020-11-15 23:58:28] 3850 downloading diff
[2020-11-15 23:59:31] 3850 filtering diff
[2020-11-15 23:59:58] 3850 importing diff
[2020-11-16 00:01:23] 3850 expiring tiles
[2020-11-16 00:01:23] 3850 Done with import
```

顯示的 `replag` 值是同步延遲；預設每次會匯入一小時份的資料。如果無法正常運作，請確認 `osmosis` 已正確安裝，且可以從命令列執行。

以相同方式再次執行 `openstreetmap-tiles-update-expire`，應該會看到 `replag` 比之前少了一小時。

接下來在 crontab 中設定 `openstreetmap-tiles-update-expire` 每隔幾分鐘執行一次。它要從負責渲染的使用者的 crontab 執行，因此在 Debian 11 以上版本需要這樣做：

```sh
sudo -u _renderd crontab -e
```

你可能會看到「problems with history file」的訊息，不用擔心。在較舊的系統上，大概可以直接以渲染圖磚的使用者身分執行 `crontab -e`。第一次執行 `crontab -e` 時，可能會要求你選擇編輯器。把游標移到檔案結尾。以 `#` 開頭的行是註解。最後一行註解是：

```sh
# m h  dom mon dow   command
```

這些欄位分別是 `m`（分鐘）、`h`（小時）、`dom`（每月的第幾日）、`mon`（月）、`dow`（星期）與 `command`（指令）。加入以下這行：

```sh
*/5  * *   *   *     /home/renderaccount/src/mod_tile/openstreetmap-tiles-update-expire
```

讓腳本每 5 分鐘執行一次。請把 `renderaccount` 改成腳本所在位置的使用者名稱。用 `tail -f /var/log/tiles/run.log` 監看輸出並等待 5 分鐘。最後它會追上進度，開放街圖上的新編輯也會出現在你的圖磚中。

## 更新可能遇到的問題

如果 `/var/log/tiles/run.log` 中出現「Expiry failed」之類的錯誤，可能是腳本中呼叫的程式之一 `render_expired` 記憶體不足。如果是這樣，請調低腳本中的 `EXPIRY_MAXZOOM`，直到可以正常運作。你可以從 `/var/log/tiles/` 中的其他檔案取得更多錯誤細節。

如果出現「rm: cannot remove '/var/lib/mod_tile/dirty_tiles.6877':」之類的錯誤，可能表示 `osm2pgsql` 執行失敗，結束時沒有產生髒圖磚清單。它的記錄檔位置定義在腳本前段，預設為 `/var/log/tiles/osm2pgsql.log`，查看這個檔案應該就能找到實際的錯誤。

## 設定 munin

如果你使用 munin 回報 `mod_tile` 與 `renderd` 的活動，可以讓它讀取 `state.txt` 檔案來顯示資料庫同步延遲：

```sh
sudo nano /etc/munin/plugins/replication_delay
```

腳本內容可以從[這裡](https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/munin/replication_delay_osmosis){: target=_blank}取得。你可能需要把其中的檔案位置改成正確的值（通常是 `/var/cache/renderd` 或 `/var/lib/mod_tile`）。然後：

```sh
sudo /etc/init.d/munin-node restart
```

完成後不久，`http://your.server.address/munin/renderd-day.html` 應該會顯示「Data import lag」圖表。如果沒有，請查看 `/var/log/munin` 中的記錄。如需進一步協助釐清問題，請參考[這裡](https://guide.munin-monitoring.org/en/latest/develop/plugins/howto-write-plugins.html){: target=_blank}。
