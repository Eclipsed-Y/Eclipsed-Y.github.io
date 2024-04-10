---
title: "单例模式为什么要使用volatile"
date: 2024-04-10T11:30:03+00:00
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

## volatile作用

1.保证可见性

2.禁止指令重排序

## 重排序

```java
int mod = 0;
int a = 1;
public void chmod(){
    a = 2;
    mod = 1
}
public void test(){
    if(mod == 1){
        System.out.println(a);
    }
}
```

由于指令重排序会导致一些前后没有直接联系的代码顺序问题，如chmod中a和mod不直接联系，可能出现mod先为1的情况，此时如果有另一线程执行test就会出问题。

故而使用volatile通过加内存屏障，可以使修饰的变量被执行时，保证之前的命令执行完毕。

## 单例模式双检锁为什么要volatile

```java
public class Single{
    private volatile static MyService service;
    private Single(){
        // 私有化构造函数
    }
    public static MyService get(){
        if(service == null){
            synchronized(Single.class){
                if(service == null){
                    service = new Single();
                }
            }
        }
        return service;
    }
}
```

如上代码，如果没有volatile，线程a执行service = new Single()时，可能导致原本的分配内存、初始化对象、分配地址改变顺序，比方说变成分配内存、分配地址、初始化对象。

这样当分配好地址后，还没初始化对象，正好b线程也进来，判断service非null，直接返回了一个未初始化的service，因此需要volatile。