---
layout: docs
title: 使用 Munin 監控
lang: zh-TW
---

# {{ title }}

"Munin" 是用來監控伺服器的 "renderd" 與 "mod_tile" 的活動。Munin 在多個平台都能使用；這份說明是採用 2022 年 6 月的 Ubuntu Linux 22.04 版本。

首先，安裝必須的軟體：

```sh
sudo apt install munin-node munin libcgi-fast-perl libapache2-mod-fcgid
```

如果您檢視 `/etc/apache2/conf-available` 您應當看到 `munin.conf` 是系統化連結到 `../../munin/apache24.conf`，實際是 `/etc/munin/apache24.conf`。

`/etc/munin/apache24.conf` 檔案是 Apache `munin` 設定檔。在檔案中，如果您想要 `munin` 能夠連結全域而非僅止是本地變動，兩樣都必須從 `Require local` 改為 `Require all granted`。

接著編輯 `/etc/munin/munin.conf`。解除以下幾行的的註解狀況：

```conf
dbdir /var/lib/munin
htmldir /var/cache/munin/www
logdir /var/log/munin
rundir /var/run/munin
```

重新啟動 munin 與 apache：

```sh
sudo /etc/init.d/munin-node restart
sudo /etc/init.d/apache2 restart
```

瀏覽 `http://yourserveripaddress/munin`。您應當看到頁面顯示 "apache"、"disk"、"munin"等等。

要在 mumin 從 mod_tile 與 renderd 增加外掛：

```sh
sudo ln -s /usr/share/munin/plugins/mod_tile* /etc/munin/plugins/
sudo ln -s /usr/share/munin/plugins/renderd* /etc/munin/plugins/
```

There should be 4 mod_tile plugins and 5 renderd ones.  Run munin's cron job manually once:

```sh
sudo -u munin munin-cron
```

Restart munin and apache again:

```sh
sudo /etc/init.d/munin-node restart
sudo /etc/init.d/apache2 restart
```

After a short delay, refreshing `http://yourserveripaddress/munin/` should now show entries for "mod_tile" and "renderd".

Munin updates its graphs every 5 minutes, as configured by the cron file `/etc/cron.d/munin`.
