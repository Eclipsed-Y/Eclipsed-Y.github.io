---
title: "docker安装redis"
date: 2024-04-03T11:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["first"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## Redis

### 安装

```sh
docker pull redis
```

### 配置

```sh
mkdir -p /home/${username}/docker/redis/conf
## 创建空白配置文件
touch /home/${username}/docker/redis/conf/redis.conf
```

### 启动

最后两行顺序不能倒

```sh
docker run \
--name redis \
-p 6379:6379 \
--restart always \
-v /home/${username}/docker/redis/data:/data \
-v /home/${username}/docker/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis \
redis-server /etc/redis/redis.conf
```

