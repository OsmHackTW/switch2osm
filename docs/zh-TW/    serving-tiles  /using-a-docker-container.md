---
layout: docs
title: 使用 Docker 容器
lang: zh-TW
---

# {{ title }}

如果您想要試試看，或是您使用 Ubuntu 以外的作業卜統，您可以採用 Docker 來容器化處理，您可以試試[這個](https://github.com/Overv/openstreetmap-tile-server){: target=_blank} (謝謝這邊所有的貢獻者)。這是依據[這邊](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md)的指引，但您可以安裝預先建構好的容器。

## Docker 容器

如果您還沒有安裝 Docker，網路已經有不少"新手指" - 參見[這邊](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10){: target=_blank}的範例

即便只是小的資料截取，您至少需要 30GB 的硬碟空間，因為全世界範圍的資料加進去也相當巨大。

## 開放街圖資料

In this example run-through I’ll download data for Zambia and import it, but any OSM .pbf file should work.  For testing, try a small .pbf first.  When logged in as the non-root user that you run Docker from, download the data for Zambia:

```sh
cd
wget https://download.geofabrik.de/africa/zambia-latest.osm.pbf
```

Create a docker volume for the data:

```sh
docker volume create osm-data
```

And install it and import the data:

```sh 
time \
    docker run \
    -v /home/renderaccount/zambia-latest.osm.pbf:/region.osm.pbf \
    -v osm-data:/data/database/ \
    overv/openstreetmap-tile-server import
```

The path to the data file needs to be the absolute path to the data file - it can't be a relative path.  In this example it's in the root directory of the "renderaccount" user.  Also, if something goes wrong, you'll need to "docker volume rm osm-data" and restart from "docker volume create osm-data" above.  At the end of the process you should see:

```sh
INFO:root: Import complete
```

If you see something like:

```sh
/data/region.osm.pbf: Is a directory
```

或是

```sh
createuser: error: creation of new role failed: ERROR: role "renderer" already exists
```

then something has gone wrong; you'll need to use "docker ps -a" to identify the failed container; "docker rm" (followed by the container id) to delete it, and then delete and recreate "osm-data" as described above.

How long this takes depends very much on the local network speed and the size of the area that you are loading. Zambia, used in this example, is relatively small.

Note that if something goes wrong the error messages may be somewhat cryptic, and unfortunately the import process can't be restarted after failure.  Also, note that newer versions of the Docker container might use newer versions of postgres, so an “osm-data” created with an earlier version might not work - you may need to remove it with “docker volume rm osm-data” and recreate.

For more details about what it’s actually doing, have a look at [this file](https://github.com/Overv/openstreetmap-tile-server/blob/master/Dockerfile){: target=_blank}. You’ll see that it closely matches the “manually building a tile server” instructions [here](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md), with some minor changes such as the tile URL and the internal account used. Internally you can see that it’s using Ubuntu 22.04.  You don’t need to interact with that directly, but you can (via "docker exec -it mycontainernumber bash") if you want to.

To start the tile server running:

```sh
docker run \
    -p 8080:80 \
    -v osm-data:/data/database \
    -d overv/openstreetmap-tile-server \
    run
```

and to check that it’s working, from a new incognito window browse to:

`http://your.server.ip.address:8080/tile/0/0/0.png`

You should see a map of the world in your browser.  Then try:

`http://your.server.ip.address:8080`

for a map that you can zoom in and out of.  Tiles (especially at low zoom levels) will take a short period of time to appear.

### 更多資訊

This docker container actually supports a lot more than the simple example here - see the [readme](https://github.com/Overv/openstreetmap-tile-server/blob/master/README.md){: target=_blank} for more details about updates, performance tweaks, etc.

### 檢視圖磚

For a simple “slippy map” that you can modify, you can use an html file “sample_leaflet.html” which is [here](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/extra/sample_leaflet.html){: target=_blank} in mod_tile’s “extra” folder. Edit “hot” in the URL in that file to read “tile”, and then just open that file in a web browser on the machine where you installed the docker container. If that isn’t possible because you’re installing on a server without a local web browser, you’ll also need to edit it to replace “127.0.0.1” with the IP address of the server and copy it to below “/var/www/html” on that server.

如果您想要載入不同區域，請重覆上述 "wget" 程序。不幸地是，每次想載入一些新資料的時候，您每次必須刪除以及重新建構 "osm-data"。
