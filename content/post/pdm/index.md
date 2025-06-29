---
title: pdm的用法
description: pdm使用心得
slug: hello pdm
date: 2025-05-30 00:00:00+0000
categories:
    - Python
tags:
    - Python
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# PDM使用
PDM是一个python包管理器，不过它**不是一个虚拟环境**，它可以将依赖安装到项目本地的__pypackages__目录，而非全局目录，从而避免污染全局环境并且实现项目隔离。

在PDM环境下，项目将优先从__pypackages__中搜索包，实现的原理就是在全局的 site-packages 目录之前加上项目目录里的__pypackages__路径，使得__pypackages__的搜索优先级高于全局的 Python 环境。

**目前我只写了一些初级的用法，后面我会随着我的深入使用更新我的博客。**

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

### 创建项目
创建项目的时候会让依次输入使用的python解释器版本、项目名、邮件等等信息，一板一眼确实挺不错的。
```
mkdir my-project && cd my-project
pdm init
```

init成功以后会在目录下创建下面的三个文件：


+ .pdm-python

    .pdm-python中存放了python解释器的路径

+ pdm.lock

    - 所有软件包及其版本
    - 包的文件名和哈希值
    - 用于下载包的源 URL
    - 每个包的依赖项和标记

```
# This file is @generated by PDM.
# It is not intended for manual editing.

[metadata]
groups = ["default"]
strategy = ["inherit_metadata"]
lock_version = "4.5.0"
content_hash = "sha256:5e86f537f04c413e1ae880b74cc0ddbde0cac75b48683e80298d9a986d7e06c5"

[[metadata.targets]]
requires_python = "==3.12.*"

```

+ pyproject.toml

    pyproject.toml中则存放了一些相关信息。

```
[project]
name = "test"
version = "0.1.0"
description = "Default template for PDM package"
authors = [
    {name = "apowerfulmei",email = "1234@qq.com"},
]
dependencies = []
requires-python = "==3.12.*"
readme = "README.md"
license = {text = "MIT"}


[tool.pdm]
distribution = false

```


### 依赖项管理
#### 增加依赖项

```
pdm add packages/localpath/url

pdm add pandas
pdm add pandas==2.2.2
pdm add ./package
pdm add "https://github.com/numpy/numpy/releases/download/v1.20.0/numpy-1.20.0.tar.gz"
pdm add "git+https://github.com/pypa/pip.git@22.0"
```

add的参数可以是包的名称，可以是本地包的路径（**本地路径要以./开头，否则会被视为普通包的命名**），可以是URL，也可以是VCS 依赖项。

增加依赖项的时候指定版本或许会是个好习惯。



#### 移除依赖项

```
pdm remove pandas
```

#### 更新依赖项

```
pdm update package
```
将依赖项更新到更新的版本。

### 配置项目

#### 更改pdm配置

```
pdm config
```

#### 下载依赖并使用pdm环境

```
pdm install
```

执行命令，pdm将下载pyproject.toml中的依赖项。

此外，配置环境的方法有两种，分别是**虚拟环境**和**PEP 582**环境：

虚拟环境下将创建一个`.venv`目录，目录中存放了下载的各种依赖包，可执行以下命令进行操作：

```
# 列举虚拟环境下的依赖项
pdm venv list

# 激活虚拟环境，test为项目名称
eval $(pdm venv activate for-test)
```

PEP 582环境将创建一个`__pypackages__`目录，目录下存放各种依赖包。在PEP 582环境下执行python命令如下：

```
pdm run python3 test.py
```
`pdm run`会做两件事：1、在执行命令前，插入 `__pypackages__` 目录到 PYTHONPATH 中；2、在执行命令后，删除 PYTHONPATH 中的 `__pypackages__` 目录。


## 一些方便的方法

### 配置已有项目
有时候我们会在创建了一个包含大量包的python项目后才想起来使用pdm进行包管理，这种情况应该怎么做呢？

可以先用pigar生成一个requirements.txt文件：

```
pip install pigar
pigar gen
```

然后直接import即可。
```
pdm import -f requirements ./requirements.txt
```

不过实践证明pigar创建的requirements所包含的依赖不一定齐全。

## 参考
https://pdm-project.org/zh-cn/latest/
https://geekdaxue.co/read/yumingmin@python/mi4af1
https://blog.csdn.net/penntime/article/details/140191708
https://www.cnblogs.com/liwenchao1995/p/17421496.html
https://zhuanlan.zhihu.com/p/492331707