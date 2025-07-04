---
title: Syzkaller Testcase Minimization
description: syzkaller测试用例最小化策略
slug: syz-Minimization
date: 2025-07-01 00:00:00+0000
categories:
    - Fuzz
tags:
    - fuzz
    - syzkaller
	- testcase minimization
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# Syzkaller Testcase Minimization

这里开一个新坑，来探讨一下syzkaller测试用例最小化是如何实现的。

## 最小化的意义

首先，为什么我们需要进行测试用例最小化。关于最小化的研究最早可以追溯到Zeller的文章[Yesterday, my program worked. Today, it does not. Why?](https://dl.acm.org/doi/10.1145/318774.318946)，文章中提出的**Delta-Debug**方法对后续的研究工作都有着极大的影响。



## Syzkaller是如何进行最小化的

Syzkaller对测试用例的最小化的主要功能集中在[prog/minimize.go文件的Minimize函数中](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/prog/minimization.go#L54)，这个函数会接收四个参数：

| 参数         | 描述 |
| -----------  | ----------- |
| p0           | 要简化的测试用例序列                   |
| callIndex0   | 目标系统调用下标                       |
| mode         | 简化模式                              |
| pred0        | 可以理解为验证函数，验证简化后的程序功能 |

函数最终将返回简化后的p0以及callIndex0。

接下来我们将具体分析该函数，可以分为两个部分来看，一个是系统调用序列的最小化，另一个是系统调用参数的最小化。

### 系统调用序列最小化

这一部分工作旨在尽可能移除无关的系统调用而仍使测试用例保留原有的功能（触发崩溃）。

[removeCalls函数](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/prog/minimization.go#L120)

随后，函数会倒序逐个移除系统调用，如果发现移除后，测试用例原功能丧失，则保留该系统调用，跳过对下一个系统调用进行测试。

```go
	for i := len(p0.Calls) - 1; i >= 0; i-- {
		if i == callIndex0 {
			continue
		}
		callIndex := callIndex0
		if i < callIndex {
			callIndex--
		}
		p := p0.Clone()
		p.RemoveCall(i) //移除克隆者的系统调用
		if !pred(p, callIndex, statMinRemoveCall, fmt.Sprintf("call %v", i)) {
			continue  //功能丧失则跳过
		}
		p0 = p 
		callIndex0 = callIndex
	}
```


## Syzkaller的最小化场景

Syzkaller最小化的场景有如下几种：Fuzz运行时、漏洞复现

### Fuzz运行时

这一部分我们可以看[job.go中的代码](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/pkg/fuzzer/job.go#L343)，从它给prog.minimize传递的pred函数可以发现，这里最小化的功能完整性体现在**经过最小化之后，目标系统调用的覆盖率并没有减小**，这说明删去的系统调用对于目标没有影响。