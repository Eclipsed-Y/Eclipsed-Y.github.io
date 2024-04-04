---
title: "VMWare Linux 虚拟机配置永久共享文件夹"
date: 2024-04-03T11:30:03+00:00
# weight: 2
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



## 设置共享文件夹

![image-20240404144045444](./imgs/image-20240404144045444.png)

## 挂载

创建挂载文件夹

```sh
mkdir /mnt/hgfs
```

挂载

```sh
/usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other -o uid=0 -o gid=0 -o umask=022
```

## 永久挂载

提升文件权限

```sh
chmod 777 /etc/fstab
```

文件尾行添加内容

```sh
echo ".host:/ /mnt/hgfs fuse.vmhgfs-fuse allow_other,uid=0,gid=0,umask=022 0 0" >> /etc/fstab
```

## 添加快捷方式

```sh
ln -s /mnt/hgfs/${文件夹名称} /home/${username}/桌面
```



