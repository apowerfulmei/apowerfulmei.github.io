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

### level7

这一关使用 `curl` 构造请求：`curl 127.0.0.1:80/complete` 。

curl直接使用时，构造的是GET请求。加上--data时，构造的是POST请求。

### level8

这一关使用python写一个小脚本。

```python
#!/usr/bin/python
import requests

response=requests.get("http://127.0.0.1:80/challenge")
print(response.content)
```

### level9-level11

这几关需要设置特殊的header，指定header中的host。Host的作用是让server知道请求的是哪一个网站，因为一个IP可能对应多个域名，server需要指导用户请求的域名到底是哪一个。

level9 使用python。

```python
#!/usr/bin/python
import requests

header = {
    "Host":"webhacking.kr:80"
}

response=requests.get("http://127.0.0.1:80/submit", headers=header)
print(response.content)

```

level10 使用curl设置header Host。

```
curl -H "Host:net-force.nl:80" 127.0.0.1:80/task
```

level11 使用netcat设置请求头。

```
echo -e "GET /fulfill HTTP/1.1\nHost: 0xf.at:80\n\n" | nc 127.0.0.1 80
```

### level12

这一关提到了一个问题，就是**路径可能是包含空格的**，这种情况下使用nc构造请求时，要将路径中的空格用编码代替将其连接起来，否则会出现解析错误。

```
echo -e "GET /progress%20request%20qualify HTTP/1.1\nHost:challenge.localhost:80\n\n" | 
nc 127.0.0.1 80
```

### level13-15

这几关提到了parameter问题，即GET请求的参数问题。使用方式也很简单，在路径末尾使用 `?para=value` 即可。

level13：`curl -H "Host:challenge.localhost:80" http://127.0.0.1:80/gate?unlock=pwgklefy`

当然，参数可能是不只一个的，这种情况下用`&`将参数分隔开即可，**注意之间不要加空格**。

level14：

```
echo -e "GET /authenticate?secure_key=jcbaywzw&private_key=miwgzszt&access_code=buadbiky HTTP/1.1\nHost:challenge.localhost:80\n\n" | nc 127.0.0.1 80
```

level15：

注意这里需要将IP用双引号括起来，因为`&`在shell中有特殊的含义，会出现错误。

```
 curl -H "Host:challenge.localhost:80" "http://127.0.0.1:80/progress?keycode=wrgcvc
sy&security_token=zlptrsug&secret_key=eectkeks"
```