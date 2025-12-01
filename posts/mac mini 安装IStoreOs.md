---
title: "mac mini 安装IStoreOs"
titleColor: "#aaa,#0ae9ad"
titleIcon: "asset:markdown"
tags: ["mac mini"]
categories: ["折腾"]
description: "mac mini 安装IStoreOs"
publishDate: "2023-03-12"
updatedDate: "2023-03-12"
password: "1231234"
---

#### 下载 UTM

[下载地址](https://cdn.jiangwei.zone/UTM.dmg)

#### 教程

[地址](https://post.smzdm.com/p/ak3eqrz9/)

#### nas-tool

[地址](https://hub.docker.com/r/nastools/nas-tools)

```sh
# docker pull nastools/nas-tools:2.9.1
docker pull jxxghp/nas-tools:latest
```

```sh
docker run -d \
--name nas-tools \
--hostname nas-tools \
-p 3000:3000 \
-v /mnt/data_vda1/nastools/:/config/ \
-v /mnt/nvme0n1-1/nastools/media:/media/ \
-v /mnt/nvme0n1-1/nastools/download:/download/ \
-e PUID=1000 \
-e PGID=1000 \
-e UMASK=000 \
-e NASTOOL_AUTO_UPDATE=false \
--restart unless-stopped \
razeencheng/nastool:2.9.1


docker run -d \
    --name nas-tools \
    --hostname nas-tools \
    -p 3000:3000 \
    -v /mnt/nvme0n1-1/nastools/config:/config \
    -v /mnt/nvme0n1-1/nastools/media:/media/local  \
    -v /mnt/CloudNAS/jw/media:/media/clouddrive \
    -e PUID=0     `# 想切换为哪个用户来运行程序，该用户的uid，详见下方说明` \
    -e PGID=0     `# 想切换为哪个用户来运行程序，该用户的gid，详见下方说明` \
    -e UMASK=000  `# 掩码权限，默认000，可以考虑设置为022` \
    -e NASTOOL_AUTO_UPDATE=false `# 如需在启动容器时自动升级程程序请设置为true` \
    razeencheng/nastool:2.9.1

docker pull razeencheng/nastool:2.9.1
```

初始账号：admin   
初始密码：password  

#### qb 
初始账号：admin    
初始密码：adminadmin   

#### TMDB
TMDB API Key：6f9dbf7c38b1073decd2bfb55b5a9320

eyJhbGciOiJIUzI1NiJ9.eyJhdWQiOiI2ZjlkYmY3YzM4YjEwNzNkZWNkMmJmYjU1YjVhOTMyMCIsIm5iZiI6MTc2MDc2NDcxNy43ODksInN1YiI6IjY4ZjMyMzJkYjE5MTY2NGZlMjA4NmZiMCIsInNjb3BlcyI6WyJhcGlfcmVhZCJdLCJ2ZXJzaW9uIjoxfQ.IyUFxzF6SVjtd_wiZJMOk6hr7kd43IxB4h5h1o8_dPo

#### flaresolverr
与Jackett联动，实现更多资源
docker pull flaresolverr/flaresolverr
docker run -d
--name=flaresolverr
-p 8191:8191
-e LOG_LEVEL=info
--restart unless-stopped
ghcr.io/flaresolverr/flaresolverr:latest

#### ChineseSubFinder
jwnoal
147258