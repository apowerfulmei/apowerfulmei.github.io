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

这一部分工作旨在尽可能移除无关的系统调用而仍使测试用例保留原有的功能（触发崩溃或维持覆盖率）。

这部分功能由[removeCalls函数](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/prog/minimization.go#L120)实现，我们可以将其分为三个部分：切割、无关系统调用剔除、遍历筛选。

**1、切割**

首先，函数会移除所有在目标系统调用之后的系统调用，因为大部分情况下，这些系统调用不会影响目标系统调用的状态。

```go
	if callIndex0 >= 0 && callIndex0+2 < len(p0.Calls) {
		// It's frequently the case that all subsequent calls were not necessary.
		// Try to drop them all at once.
		p := p0.Clone()
		for i := len(p0.Calls) - 1; i > callIndex0; i-- {
			p.RemoveCall(i)
		}
		if pred(p, callIndex0, statMinRemoveCall, "trailing calls") {
			p0 = p
		}
	}
```

**2、无关系统调用剔除**

随后，将调用函数[removeUnrelatedCalls](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/prog/minimization.go#L160)，移除和目标系统调用无关的系统调用。

它首先调用函数relatedCalls，生成一个 `map[int]bool` 类型字典，字典表示每个系统调用和目标系统调用是否有关，随后遍历整个测试用例，将无关的系统调用删去，并测试功能是否完整。

relatedCalls函数会首先调用 `uses` 函数：

```go
func uses(call *Call) map[any]bool {
	used := make(map[any]bool)
	// 遍历所有的参数
	ForeachArg(call, func(arg Arg, _ *ArgCtx) {
		switch typ := arg.Type().(type) {
		case *ResourceType:
			a := arg.(*ResultArg) 
			used[a] = true
			if a.Res != nil {
				used[a.Res] = true
			}
			// args that use this arg
			for use := range a.uses {
				used[use] = true
			}
		case *BufferType:
			a := arg.(*DataArg)
			if a.Dir() != DirOut && typ.Kind == BufferFilename {
				val := string(bytes.TrimRight(a.Data(), "\x00"))
				used[val] = true
			}
		}
	})
	return used
}
```

提取出目标系统调用使用的所有资源，将其存放在used map中。随后遍历所有其他系统调用，同样提取其使用的资源used1，将其与used进行比较，如果used1使用了used中使用过的资源，则认为该系统调用是**相关**的，并将used1合并到used中。


**3、遍历筛选**

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

### 系统调用参数最小化

这一部分改天更新。


## Syzkaller的最小化场景

Syzkaller最小化的场景有如下几种：Fuzz运行时、崩溃复现

### Fuzz运行时

这一部分我们可以看[job.go中的代码](https://github.com/google/syzkaller/blob/a55c0058ebb614a710cb236c1b47f30aeb88ab12/pkg/fuzzer/job.go#L343)，从它给prog.minimize传递的pred函数可以发现，这里最小化的功能完整性体现在**经过最小化之后，目标系统调用的覆盖率并没有减小**。

```go
	p, call := prog.Minimize(job.p, call, mode, func(p1 *prog.Prog, call1 int) bool {
		if stop {
			return false
		}
		var mergedSignal signal.Signal
		for i := 0; i < minimizeAttempts; i++ {
			...
			if !reexecutionSuccess(result.Info, info.errno, call1) {
				// The call was not executed or failed.
				continue
			}
			// 获取目标call执行后的signal
			thisSignal := getSignalAndCover(p1, result.Info, call1)
			if mergedSignal.Len() == 0 {
				mergedSignal = thisSignal
			} else {
				mergedSignal.Merge(thisSignal)
			}
			// signal没有发生变化，signal的长度相等
			if info.newStableSignal.Intersection(mergedSignal).Len() == info.newStableSignal.Len() {
				...
				return true
			}
		}
		...
		return false
	})
```

系统调用与系统调用之间存在着隐式与显式的依赖关系，一个系统调用可能会对另一个系统调用产生一些影响，使其执行后的覆盖率发生变化，经过最小化处理后，测试用例会删去那些对目标系统调用无关的系统调用。


### 崩溃复现

这一部分内容我们可以参考syz-repro工具的原理，在一个测试用例触发崩溃之后，syzkaller会进行最小化，提取出能出发崩溃的最小测试用例。

在实现上和上一个流程基本相同，但在调用Minimize函数时，会将callIndex设置为-1，这将跳过切割与无关剔除步骤，直接遍历每个系统调用，观察删去该系统调用之后，崩溃是否还能成功触发。