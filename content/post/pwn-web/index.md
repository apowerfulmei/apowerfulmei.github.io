---
title: pwn.college web
description: pwn web相关部分的记录
slug: hello pwn web
date: 2025-06-30 00:00:00+0000
categories:
    - PWN
tags:
    - PWN
    - Web
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# PWN Web

我将在这里记录pwn.college中和web相关部分的练习。


## Playing With Programs - Talking Web

这一章节涉及到的是一些web相关的程序以及命令的基础用法，为后面进阶的内容打基础。

### level1 - level4

这部分内容比较简单，就跳过了。

### level5 - level6

这一关用到了`netcat`，也就是`nc`。netcat提供了一个和server交互的窗口用于构造请求。

首先连接server：`nc 127.0.0.1 80`，连接完成以后，按照如下格式构造请求：`GET / HTTP/1.1`。分别代表请求类型，路径，协议。随后netcat会自动构造完整的请求。

level6 同理，只不过多了一个/verify路径：`GET /verify HTTP/1.1` 。整个命令可以用一行表示 `echo -e "GET /verify HTTP/1.1\n" | nc 127.0.0.1 80`。

![level6](level6-nc.png)