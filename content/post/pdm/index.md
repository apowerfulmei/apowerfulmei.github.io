---
title: pdm的用法
description: pdm使用心得
slug: hello pdm
date: 2025-05-29 00:00:00+0000
categories:
    - Python
tags:
    - Python
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# PDM使用

## 下载与安装

使用pip下载，可能会出先网络连接不成功，下载失败的情况，这种情况指定国内源即可：
```
pip install pdm
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pdm

```

## 使用

### pdm install速度慢处理
指定pdm使用国内源：
```

pdm config pypi.url https://pypi.tuna.tsinghua.edu.cn/simple

```

## 参考
https://geekdaxue.co/read/yumingmin@python/mi4af1
https://blog.csdn.net/penntime/article/details/140191708
https://www.cnblogs.com/liwenchao1995/p/17421496.html