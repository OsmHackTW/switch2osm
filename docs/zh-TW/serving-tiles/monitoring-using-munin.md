---
layout: docs
title: 使用 Munin 監測
lang: zh-TW
---

# {{ title }}

「Munin」可以用來監測伺服器上「renderd」與「mod_tile」的活動。Munin 支援多種平台，以下說明於 2022 年 6 月在 Ubuntu Linux 22.04 上測試。

首先安裝必要的軟體：

```sh
sudo apt install munin-node munin libcgi-fast-perl libapache2-mod-fcgid
```

查看 `/etc/apache2/conf-available`，你應該會看到 `munin.conf` 是指向 `../../munin/apache24.conf`（也就是 `/etc/munin/apache24.conf`）的符號連結。

`/etc/munin/apache24.conf` 檔案是 Apache 的 `munin` 設定檔。如果希望從任何地方都能存取 `munin`，而不限於本機，請把檔案中的兩處 `Require local` 都改成 `Require all granted`。

接著編輯 `/etc/munin/munin.conf`，取消以下幾行的註解：

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

前往 `http://yourserveripaddress/munin`，應該會看到列出「apache」、「disk」、「munin」等項目的頁面。

把 mod_tile 與 renderd 的外掛加入 munin：

```sh
sudo ln -s /usr/share/munin/plugins/mod_tile* /etc/munin/plugins/
sudo ln -s /usr/share/munin/plugins/renderd* /etc/munin/plugins/
```

應該會有 4 個 mod_tile 外掛與 5 個 renderd 外掛。手動執行一次 munin 的 cron 工作：

```sh
sudo -u munin munin-cron
```

再次重新啟動 munin 與 apache：

```sh
sudo /etc/init.d/munin-node restart
sudo /etc/init.d/apache2 restart
```

稍待片刻後，重新整理 `http://yourserveripaddress/munin/`，應該就會看到「mod_tile」與「renderd」的項目。

Munin 每 5 分鐘更新一次圖表，由 cron 檔案 `/etc/cron.d/munin` 設定。
