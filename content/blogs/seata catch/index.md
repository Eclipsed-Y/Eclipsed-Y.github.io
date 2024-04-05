---
title: "全局异常捕获器令Seata事务无法回滚的解决方案"
date: 2020-09-14T11:30:03+00:00
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

## 起因

项目使用seata进行分布式事务管理

项目使用了全局异常处理器来捕获异常



服务A执行分布式事务：

1.A中方法

2.用Feign调用B中方法，方法抛出异常

最终1.成功提交（未回滚），2.未提交

## 原因

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 捕获业务异常
     * @param ex
     * @return
     */
    @ExceptionHandler
    public Result<?> exceptionHandler(BaseException ex){
        log.error("异常信息：{}", ex.getMessage());
        BaseContext.removeCurrentId();
        return Result.error(ex.getMessage());
    }
}
```

全局异常处理器捕获异常，导致A中方法得到Feign调用的正常返回值，因此Seata认为无异常

## 解决

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 捕获业务异常
     * @param ex
     * @return
     */
    @ExceptionHandler
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<?> exceptionHandler(BaseException ex){
        log.error("异常信息：{}", ex.getMessage());
        BaseContext.removeCurrentId();
        return Result.error(ex.getMessage());
    }
}
```

通过添加响应状态码为4xx，5xx，即可令Feign调用返回异常